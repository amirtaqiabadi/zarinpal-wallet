# Payment flow

ZarinPal operation names below are roles, not URLs. Resolve the path, the success `Status` value, and the verify codes with `search_docs`, `get_doc`, and `get_error_code` before coding. The codes `100` (first verify) and `101` (already verified) are the expected verify outcomes; confirm both against the live error-code tool and branch on the documented values.

Do the HTTP call outside the row lock. The database transaction starts at the lock step.

## 1. Initiate

Authenticated. `initiatorId` is the session user.

1. Validate amount as a positive integer rial, at least the live minimum.
2. Open a database transaction.
3. Resolve the team as in [schema.md](schema.md): active owner of the given team, or create a personal wallet.
4. Insert `gateway_attempts`:
   - `type = zarinpal`
   - `transaction_type = gateway_credit`
   - `status = pending` (`10-pending` in the database)
   - `amount` as requested
   - `fee = 0`
   - `salt` from a 32-byte CSPRNG, Base64
   - `merchant_correlation_key` 64-character Crockford Base32
   - `initiator_id`, `team_id`
5. Set `callback_url` to the HMAC URL in [SKILL.md](../SKILL.md). Commit is still open.
6. Call the ZarinPal payment-request operation with the stored amount, that `callback_url`, a short description, and the user's mobile when the host has one. Currency `IRR`.
7. If the call fails or returns no authority: set this row to `failed`, store the gateway code in `metadata` without secrets, commit, and return `GATEWAY_REJECTED` or `GATEWAY_UNAVAILABLE`.
8. If it succeeds: set `session_token` to the authority and `fee` from the response. Merge metadata; do not drop keys you already stored. Commit.
9. Respond with the payment URL from the live docs. The browser opens that URL. Do not embed the token in a query string of your own API.

A pending row with no authority is never credited.

## 2. Callback

No session. Exempt from CSRF only after the HMAC check. Accept the HTTP method the live docs use for the return redirect.

1. Read `sig` from the query string. Read the authority and status fields using the names from the live docs (`Authority` and `Status` unless the page says otherwise).
2. Load the attempt by `session_token = authority`. If none, redirect to the host's failure page with `ATTEMPT_NOT_FOUND`. Do not create a row.
3. Validate HMAC against this row. On failure, redirect with `INVALID_CALLBACK_SIGNATURE` and change nothing.
4. If status is not the documented success value: lock the attempt `WHERE session_token = $authority AND status = '10-pending' FOR UPDATE`. If the lock finds a row, set `failed` and merge the gateway status into metadata. Do not call verify. Do not insert a ledger row.
5. If status is success, call verify with `merchant_id`, the authority, and `amount` copied from this database row.

Then, inside a new database transaction:

6. Lock `WHERE session_token = $authority AND amount = $storedAmount FOR UPDATE`.
7. If no row, stop with `ATTEMPT_NOT_FOUND`. If verify reported an amount and it differs from the locked row, do not credit. Mark that attempt `failed` only when its status is still `10-pending`, then stop with `AMOUNT_MISMATCH`.
8. If verify code is the first-success code or the already-verified code:
   - When a `gateway_credit` row already exists for this attempt, commit any missing `successful` status and stop. Do not add balance.
   - Otherwise set `successful`, `reference_id` to `ref_id`, `masked_pan` to masked `card_pan`, `hashed_pan` to `card_hash`, and `fee` when the payload includes it. Merge metadata; keep the original keys.
   - Lock the latest ledger row for `team_id`.
   - Insert `transactions`: `type = gateway_credit`, `amount` = attempt amount, `available_balance_after` = previous available + amount, `restricted_balance_after` = previous restricted (or `0`).
9. Any other verify code: do not credit. Mark `failed` only while status is `10-pending`.

Redirect the browser to the host success or failure page. Put a stable error code in the query, not a stack or a PAN.

## 3. Reverse

Authenticated owner of the attempt's team. CSRF required.

Eligible only when all of these hold:

- the ledger row is `gateway_credit`
- the attempt `transaction_type` is `gateway_credit`
- the credit's `created_at` is under 49 minutes ago
- `reference_id` is present
- attempt status is `50-successful`
- no `reverse` row exists for this attempt
- latest available balance is greater than or equal to the attempt amount

Inside one database transaction:

1. Lock the attempt by id. Re-check status is `successful`.
2. Re-check that no `reverse` row exists.
3. Lock the latest ledger row. If available balance is too low, throw `INSUFFICIENT_BALANCE` and roll back.
4. Insert `transactions` with `type = reverse`, `amount = 0 - attempt.amount`, and `available_balance_after` reduced by that amount. Leave `restricted_balance_after` unchanged. Do not delete or update the credit row.
5. Call the ZarinPal reversal or refund operation from the live docs, using the authority and `ref_id`. Do not call SEP.

Result:

- Documented success: set attempt status to `reversed` and commit.
- No response (timeout or network): set `reverse_pending`, merge a non-sensitive note into metadata, and commit. The reverse ledger row stays.
- Documented rejection: throw `GATEWAY_REJECTED` so the database transaction rolls back. The credit remains and no reverse row is kept.

## 4. Refund request

A refund request is a row in `refund_requests`, not a second gateway credit. Create one only when the user asks for a manual card refund. `card_number` is 16 digits. One request per `transaction_id`. Do not log the card. List responses show the first 6 and last 4 digits only.

This does not replace reverse. Reverse is the gateway void path above.
