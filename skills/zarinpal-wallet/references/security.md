# Security rules

These rules are mandatory. They come from the wallet payment path this skill is based on: atomic initiate, HMAC on the unauthenticated callback, row locks, verify-before-credit, and a ledger that only appends.

## Identity and CSRF

- `initiatorId` comes from the authenticated session. Ignore any initiator id in the body.
- Charging a team requires an active owner membership. See [schema.md](schema.md).
- State-changing routes use double-submit CSRF: cookie `csrf-token`, header `x-csrf-token`, compared with timing-safe equality. Cookie is `SameSite=Strict`.
- The payment callback is the only payment route exempt from CSRF, and only because HMAC is checked first. Do not exempt initiate or reverse.
- Session cookies are `Secure` and `HttpOnly` in production.

## Secrets and logs

- `ZARINPAL_MERCHANT_ID`, `PAYMENT_CALLBACK_HMAC_SECRET`, and `ZARINPAL_SANDBOX` come from the environment.
- Sandbox sends a generated UUID as `merchant_id` and talks only to `https://sandbox.zarinpal.com`. After registration, replace that UUID with the merchant id ZarinPal issued and set `ZARINPAL_SANDBOX=false`. Do not send the sandbox UUID to `https://payment.zarinpal.com`.
- Never log the merchant id, authority, HMAC, salt, callback secret, or a full card number.
- Never put the merchant id or the authority in a query string you control, except the authority inside the payment URL the live ZarinPal docs tell you to open.
- Gateway attempts store masked PAN and card hash only.
- Errors returned to clients have no stack trace. Log the internal failure with the request id on the server.

## HMAC

Generate a 32-byte salt with a CSPRNG and store it Base64 on the attempt.

```text
data   = "{attemptId}:{amount}:{salt}"
digest = HMAC_SHA256(PAYMENT_CALLBACK_HMAC_SECRET, data) as Base64
```

`amount` is the decimal form of the stored bigint, with no separators. Put `sig` on the callback URL you send to ZarinPal.

On callback, load the attempt, then compare `sig` to the digest of that row's id, amount, and salt. Use timing-safe equality on the raw digest bytes. If the lengths differ, reject. Missing salt, missing `sig`, or a mismatch stops the flow: no verify, no status change, no credit.

The callback URL is built on the server. Reject a client-supplied callback URL.

## Amount

- Amount is an integer rial of at least 1000. The fee page documents that floor; error `-9` also means the amount is outside the allowed range.
- The verify request sends the amount stored on the attempt. Never send an amount from the callback, the client, or a fee-inclusive suggestion.
- If the verify code is `-50`, or the verify payload includes an amount and it differs from the locked row, do not credit. Mark the attempt failed only when its status is still pending.
- Currency sent to ZarinPal is `IRR`. Do not send `IRT`. Do not mix rial and toman in one attempt.

## Concurrency

Do not hold a row lock during the HTTP call to ZarinPal.

Credit and reverse run inside a database transaction:

1. `SELECT ... FROM gateway_attempts WHERE session_token = $authority AND amount = $storedAmount FOR UPDATE`.
2. Apply the status rules below.
3. `SELECT available_balance_after, restricted_balance_after FROM transactions WHERE team_id = $teamId ORDER BY created_at DESC LIMIT 1 FOR UPDATE`.
4. Insert the new ledger row.

Use parameterized SQL only. The unique key `(type, gateway_attempt_id)` is the backstop against a second `gateway_credit` or a second `reverse` for the same attempt.

Status transitions:

- Credit only from `10-pending` or from `50-successful` when the `gateway_credit` row is missing (crash after verify, before insert).
- If `50-successful` and the credit row exists, return success and stop.
- Failure updates only a `10-pending` row. Never mark `50-successful` as failed.
- A successful attempt is reversed only from `50-successful`, and only when no `reverse` row exists.

`merchant_correlation_key` is 64 characters of Crockford Base32 (`0123456789ABCDEFGHJKMNPQRSTVWXYZ`) from a CSPRNG.

## HTTP hardening

- Rate-limit payment initiate and the callback.
- Cap JSON body size (10 MB unless the host already uses a stricter limit).
- Assign a request id and copy it to `traceId`.
- Parameterized queries only.
- Hide the server implementation banner in production.

## Error codes

| code | type | when |
| --- | --- | --- |
| `AMOUNT_INVALID` | `VALIDATION_ERROR` | missing, non-integer, or below 1000 rials |
| `DESCRIPTION_INVALID` | `VALIDATION_ERROR` | missing, or longer than 500 characters |
| `TEAM_FORBIDDEN` | `DOMAIN_ERROR` | caller is not an active owner of `teamId` |
| `INVALID_CALLBACK_SIGNATURE` | `DOMAIN_ERROR` | HMAC missing or mismatch |
| `ATTEMPT_NOT_FOUND` | `DOMAIN_ERROR` | no row for this authority and amount |
| `PAYMENT_NOT_VERIFIED` | `DOMAIN_ERROR` | verify did not return the success code |
| `AMOUNT_MISMATCH` | `DOMAIN_ERROR` | verify amount differs from the stored amount |
| `INSUFFICIENT_BALANCE` | `DOMAIN_ERROR` | reverse would take available balance below zero |
| `REVERSE_NOT_ALLOWED` | `DOMAIN_ERROR` | not a `gateway_credit`, or attempt is not successful |
| `REVERSE_WINDOW_EXPIRED` | `DOMAIN_ERROR` | older than 30 minutes |
| `ALREADY_REVERSED` | `DOMAIN_ERROR` | a `reverse` row already exists |
| `GATEWAY_UNAVAILABLE` | `SYSTEM_ERROR` | request call failed before an authority existed |
| `GATEWAY_REJECTED` | `SYSTEM_ERROR` | ZarinPal returned a documented rejection |

Messages are Persian and, when a user can act, say what to do. Example: `موجودی کافی برای برگشت تراکنش نیست. کیف پول را شارژ کنید.`
