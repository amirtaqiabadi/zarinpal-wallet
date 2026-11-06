# Payment flow

Paths and codes below were checked with `zarinpal-docs`. If `get_doc` or `get_error_code` disagrees, follow the tool. `100` is the first success. `101` means the payment was already verified.

Do the HTTP call outside the row lock. The database transaction starts at the lock step.

## 1. Initiate

Authenticated. `initiatorId` is the session user.

1. Validate amount as an integer rial of at least 1000 (`AMOUNT_INVALID` otherwise). Reject an empty description or one longer than 500 characters (`DESCRIPTION_INVALID`) before calling ZarinPal. `-9` is the gateway's validation error for the same fields.
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
6. `POST` `{host}/pg/v4/payment/request.json` with `merchant_id`, the stored `amount`, `currency` `IRR`, `description`, `callback_url`, and `metadata.auto_verify` `false`. Add `metadata.mobile` when the host has one. Do not send `referrer_id`. The host is `https://sandbox.zarinpal.com` when `ZARINPAL_SANDBOX` is true, and `https://payment.zarinpal.com` when it is false. See [SKILL.md](../SKILL.md).
7. If the call fails, `data.code` is not `100`, no authority is returned, the sandbox authority does not start with `S`, or a production authority starts with `S`: set this row to `failed`, store the gateway code in `metadata` without secrets, commit, and return `GATEWAY_REJECTED` or `GATEWAY_UNAVAILABLE`. Do not redirect.
8. If it succeeds: set `session_token` to the authority and `fee` from the response. Merge metadata; do not drop keys you already stored. Commit.
9. Respond with `{host}/pg/StartPay/{authority}`. The browser opens that URL. Do not embed the token in a query string of your own API.

A pending row with no authority is never credited.

## 2. Callback

No session. Exempt from CSRF only after the HMAC check. ZarinPal redirects the browser back with `Authority` and `Status` on the query string.

1. Read `sig`, `Authority`, and `Status` from the query string.
2. Load the attempt by `session_token = authority`. If none, redirect to the host's failure page with `ATTEMPT_NOT_FOUND`. Do not create a row.
3. Validate HMAC against this row. On failure, redirect with `INVALID_CALLBACK_SIGNATURE` and change nothing.
4. If `Status` is not `OK`: lock the attempt `WHERE session_token = $authority AND status = '10-pending' FOR UPDATE`. If the lock finds a row, set `failed` and merge the gateway status into metadata. Do not call verify. Do not insert a ledger row. `NOK` means the buyer cancelled or the payment failed.
5. If `Status` is `OK`, `POST` `{host}/pg/v4/payment/verify.json` with `merchant_id`, the authority, and `amount` copied from this database row. Use the same host as the payment request. Do not send a fee or a suggested amount. In production, an authority that starts with `S` skips verify and does not credit.

Then, inside a new database transaction:

6. Lock `WHERE session_token = $authority AND amount = $storedAmount FOR UPDATE`.
7. If no row, stop with `ATTEMPT_NOT_FOUND`. If verify code is `-50`, or verify reported an amount and it differs from the locked row, do not credit. Mark that attempt `failed` only when its status is still `10-pending`, then stop with `AMOUNT_MISMATCH`.
8. If verify code is `100` or `101`:
   - When a `gateway_credit` row already exists for this attempt, commit any missing `successful` status and stop. Do not add balance.
   - Otherwise set `successful`, `reference_id` to `ref_id`, `masked_pan` to masked `card_pan`, `hashed_pan` to `card_hash`, and `fee` when the payload includes it. Merge metadata; keep the original keys.
   - Lock the latest ledger row for `team_id`.
   - Insert `transactions`: `type = gateway_credit`, `amount` = attempt amount, `available_balance_after` = previous available + amount, `restricted_balance_after` = previous restricted (or `0`).
9. Any other verify code, including `-51`, `-53`, and `-54`: do not credit. Mark `failed` only while status is `10-pending`.

Redirect the browser to the host success or failure page. Put a stable error code in the query, not a stack or a PAN.

## 3. Reverse

Authenticated owner of the attempt's team. CSRF required.

Eligible only when all of these hold:

- the ledger row is `gateway_credit`
- the attempt `transaction_type` is `gateway_credit`
- the credit's `created_at` is under 30 minutes ago
- `reference_id` is present
- attempt status is `50-successful`
- no `reverse` row exists for this attempt
- latest available balance is greater than or equal to the attempt amount

Inside one database transaction:

1. Lock the attempt by id. Re-check status is `successful`.
2. Re-check that no `reverse` row exists.
3. Lock the latest ledger row. If available balance is too low, throw `INSUFFICIENT_BALANCE` and roll back.
4. Insert `transactions` with `type = reverse`, `amount = 0 - attempt.amount`, and `available_balance_after` reduced by that amount. Leave `restricted_balance_after` unchanged. Do not delete or update the credit row.
5. `POST` `{host}/pg/v4/payment/reverse.json` with `merchant_id` and `authority` only, on the same host as the original payment. Do not send `ref_id`. Do not call SEP.

Result:

- `data.code` `100`: set attempt status to `reversed` and commit.
- No response (timeout or network): set `reverse_pending`, merge a non-sensitive note into metadata, and commit. The reverse ledger row stays.
- A documented rejection (`-60`, `-61`, `-62`, `-63`, or any other reverse error): throw `GATEWAY_REJECTED` so the database transaction rolls back. The credit remains and no reverse row is kept. `-62` means the terminal IP is not set. `-63` means the 30-minute window has expired.

## 4. Refund request

A refund request is a row in `refund_requests`, not a second gateway credit. Create one only when the user asks for a manual card refund. `card_number` is 16 digits. One request per `transaction_id`. Do not log the card. List responses show the first 6 and last 4 digits only.

This does not replace reverse. Reverse is the gateway void path above.

## 5. Missed callback

`POST` `{host}/pg/v4/payment/unVerified.json` with `merchant_id` when a pending attempt never receives a callback. The list holds at most the last 100 paid authorities that were not verified, each with an `amount`. For an authority that matches a pending attempt of the same amount, run the verify and credit steps in section 2. Do not insert a ledger row from the list itself.

`POST` `{host}/pg/v4/payment/inquiry.json` only answers with a status. Do not credit from it.
