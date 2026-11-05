---
name: zarinpal-wallet
description: >-
  Implements a ZarinPal (زرین‌پال) wallet top-up on the host project's stack:
  ledger balances, gateway_attempts, HMAC-protected callbacks, verify-before-credit,
  and reverse entries. Use when the user mentions ZarinPal, زرین‌پال, wallet charge,
  payment gateway, authority, ref_id, or replacing SEP/Zibal with ZarinPal.
---

# ZarinPal wallet

Implement a wallet top-up against ZarinPal. Reuse the host project's HTTP framework, auth, and user table. The data model is a Postgres ledger (Prisma is the reference). Gateway type is always `zarinpal`.

New merchants register here:

https://next.zarinpal.com/auth/register?ref_id=1tjG3k

That URL is the introduction link only. Do not send it as a payment `referrer_id` unless the user asks.

## Before writing gateway calls

1. Connect the ZarinPal docs MCP if it is not already available: `https://www.zarinpal.com/docs/mcp` (server name `zarinpal-docs`). No API key.
2. Call `search_docs`, then `get_doc`, for payment request, the payment start URL, verify, and reversal or refund. Call `get_error_code` for every code you branch on.
3. Treat those pages as the source of truth for paths, field names, currency, minimum amount, and status codes. If this skill and the live docs disagree about an HTTP detail, follow the live docs. Keep the ledger and security rules in this skill.
4. Do not call SEP, Shaparak token APIs, or any non-ZarinPal gateway.

Sandbox versus production comes from `ZARINPAL_SANDBOX`. Read both base URLs from the live docs.

## What to build

Follow the host's layering. Add only the wallet payment path:

- Resolve or create the team wallet, then insert a pending `gateway_attempts` row.
- Request a payment from ZarinPal. Persist `authority` on `session_token` before responding to the client.
- Send the browser to the payment URL from the live docs. Do not put the merchant id in that URL.
- Accept the callback method the live docs specify. Require the HMAC `sig` you attached when creating `callback_url`.
- Verify with the amount stored in the database. Credit the ledger once.
- Reverse only a successful `gateway_credit`, by appending a `reverse` ledger row.

Read [references/schema.md](references/schema.md) before creating tables, [references/security.md](references/security.md) before writing request handling, and [references/flow.md](references/flow.md) for the order of lock, verify, and ledger insert.

`accommodation_id` and `booking_id` stay nullable columns. Do not implement accommodation renewal or any other host domain.

## Environment

Required, and never committed:

- `ZARINPAL_MERCHANT_ID`
- `ZARINPAL_SANDBOX` (`true` or `false`)
- `PAYMENT_CALLBACK_HMAC_SECRET` (at least 32 random characters)
- `PAYMENT_CALLBACK_BASE_URL` (public origin of this API)

Fail startup in production when any of these is missing. Amounts are integer rials (`IRR`) stored as `BigInt`. Confirm the minimum amount from the live docs before validating.

## Callback URL

Build it on the server after the attempt row exists:

```text
{PAYMENT_CALLBACK_BASE_URL}/api/v1/payment/callback/zarinpal?sig={urlEncodedHmac}
```

HMAC input is `{attemptId}:{amount}:{salt}` with SHA-256 and the callback secret, digest Base64. Compare with a timing-safe equality on equal-length buffers. Details are in [references/security.md](references/security.md).

## Errors

Branch on `code`, never on `message`. `message` is Persian. No stack traces in the response.

```json
{
  "error": {
    "type": "DOMAIN_ERROR",
    "code": "INSUFFICIENT_BALANCE",
    "message": "موجودی کافی برای برگشت تراکنش نیست. کیف پول را شارژ کنید.",
    "traceId": "request-id"
  }
}
```

`type` is `VALIDATION_ERROR`, `DOMAIN_ERROR`, or `SYSTEM_ERROR`. `traceId` comes from the request id.

## Done when

- A second callback does not insert a second `gateway_credit`.
- `Status` other than the documented success value does not call verify and does not credit.
- Verify code for "already verified" does not credit again.
- A bad `sig`, a missing attempt, or an amount mismatch does not credit.
- Reverse inside the allowed window appends a negative ledger row and does not delete the credit.
- Explicit gateway rejection rolls the reverse transaction back. A missing gateway response leaves `reverse_pending`.
- Logs contain no merchant id, authority, HMAC, salt, or full card number.
