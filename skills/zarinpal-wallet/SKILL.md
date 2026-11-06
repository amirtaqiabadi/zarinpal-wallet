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
3. Treat those pages as the source of truth for paths, field names, currency, minimum amount, and status codes. The contract in [Gateway contract](#gateway-contract) was checked against that MCP. If `get_doc` or `get_error_code` disagrees with it, follow the tool. Keep the ledger and security rules in this skill.
4. Do not call SEP, Shaparak token APIs, or any non-ZarinPal gateway.

Start on the sandbox. Production is the host from the live docs, and only after the merchant replaces the sandbox merchant id with the one ZarinPal issued. The switch is in [Sandbox first](#sandbox-first).

## What to build

Follow the host's layering. Add only the wallet payment path:

- Resolve or create the team wallet, then insert a pending `gateway_attempts` row.
- Request a payment from ZarinPal on the sandbox host until cutover. Persist `authority` on `session_token` before responding to the client.
- Send the browser to the payment URL on that same host. Do not put the merchant id in that URL.
- Accept the callback method the live docs specify. Require the HMAC `sig` you attached when creating `callback_url`.
- Verify with the amount stored in the database. Credit the ledger once.
- Reverse only a successful `gateway_credit`, by appending a `reverse` ledger row.

Read [references/schema.md](references/schema.md) before creating tables, [references/security.md](references/security.md) before writing request handling, and [references/flow.md](references/flow.md) for the order of lock, verify, and ledger insert.

`accommodation_id` and `booking_id` stay nullable columns. Do not implement accommodation renewal or any other host domain.

## Sandbox first

Ship against the sandbox. No money moves until the merchant registers and pastes in the credentials ZarinPal assigns. Source: [سرویس تست (sandbox)](https://www.zarinpal.com/docs/paymentGateway/sandBox.html).

`ZARINPAL_SANDBOX` defaults to `true`. An unset value stays sandbox. In the host's production environment, an unset value fails startup; the operator sets `false` only in the cutover below.

While `ZARINPAL_SANDBOX=true`:

- Keep the path from the live docs and change the host. `https://payment.zarinpal.com/...` becomes `https://sandbox.zarinpal.com/...`. Do this for payment request, verify, the StartPay redirect, and reversal or refund. A mix of sandbox and production hosts makes ZarinPal return an unexpected error.
- `ZARINPAL_MERCHANT_ID` is any UUID you generate (`xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx`). A registered merchant is not required. Put it only in the environment.
- The sandbox checkout does not charge the card.
- Every authority the sandbox returns starts with `S`. If the payment-request body has no authority, or the authority does not start with `S`, mark the attempt `failed` and do not redirect the browser.

After the user registers at the introduction link and ZarinPal shows their merchant id:

- Set `ZARINPAL_SANDBOX=false`.
- Replace `ZARINPAL_MERCHANT_ID` with the 36-character merchant id ZarinPal assigned. Remove the sandbox UUID from that environment.
- Call `https://payment.zarinpal.com/...` again, with the paths from the live docs.
- A production authority starts with `A`. Do not verify or credit an authority that starts with `S` while sandbox is off.
- In the gateway settings, set this server's IP. Without that limit, a request can return `-10` and reverse returns `-62`.

## Gateway contract

Payment calls are `POST` JSON. Do not use the GraphQL API under `/apiDocs` to request, verify, or reverse a payment.

| Call | Path |
| --- | --- |
| Request | `/pg/v4/payment/request.json` |
| StartPay | `/pg/StartPay/{authority}` |
| Verify | `/pg/v4/payment/verify.json` |
| Reverse | `/pg/v4/payment/reverse.json` |
| Unverified list | `/pg/v4/payment/unVerified.json` |
| Inquiry | `/pg/v4/payment/inquiry.json` |

The host is `https://sandbox.zarinpal.com` or `https://payment.zarinpal.com`, from [Sandbox first](#sandbox-first).

Request body: `merchant_id`, integer `amount` in rials (minimum 1000), `currency` `IRR`, `description` (required, at most 500 characters), and `callback_url`. Send `metadata.mobile` when the host has one. Send `metadata.auto_verify` `false` so the panel cannot settle the payment before this service calls verify. Do not send `referrer_id` unless the user asks.

Use the authority only when `data.code` is `100`. The callback redirect puts `Authority` and `Status` on the query string. Verify only when `Status` is `OK`. `NOK` fails the attempt: no verify, no credit.

Verify sends `merchant_id`, the stored authority, and the stored amount. Do not send a fee-adjusted or suggested amount. `100` is the first verify. `101` means already verified: credit only when this attempt has no `gateway_credit` row. `-50` is an amount mismatch. `-51`, `-53`, and `-54` do not credit.

Reverse sends `merchant_id` and `authority` only. Do not send `ref_id`. The gateway allows it for 30 minutes after payment (`-63` after that). Success is `data.code` `100`. `-60`, `-61`, `-62`, and `-63` are rejections.

`inquiry` returns a status (`VERIFIED`, `PAID`, `IN_BANK`, `FAILED`, `REVERSED`) and never credits. `unVerified` lists at most the last 100 paid-but-unverified authorities. A missed callback is recovered by verifying a matching attempt through the same lock-and-credit path.

`-9` is a validation failure: missing merchant id, callback, or description, a description over 500 characters, an amount outside the allowed range, or a bad `referrer_id`.

## Environment

Required, and never committed:

- `ZARINPAL_MERCHANT_ID` — any UUID while sandbox is on; the merchant id ZarinPal issued after registration when sandbox is off
- `ZARINPAL_SANDBOX` (`true` or `false`; unset means `true` outside production)
- `PAYMENT_CALLBACK_HMAC_SECRET` (at least 32 random characters)
- `PAYMENT_CALLBACK_BASE_URL` (public origin of this API)

Fail startup in production when any of these is missing, and when `ZARINPAL_SANDBOX` is not exactly `true` or `false`. Amounts are integer rials (`IRR`) stored as `BigInt`. The documented minimum is 1000 rials.

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

- With `ZARINPAL_SANDBOX=true`, request, verify, StartPay, and reversal all use `https://sandbox.zarinpal.com`, the merchant id is a UUID, and a stored authority starts with `S`.
- With `ZARINPAL_SANDBOX=false`, those calls use `https://payment.zarinpal.com` and `ZARINPAL_MERCHANT_ID` is the value ZarinPal issued after registration. An authority that starts with `S` is not verified or credited.
- A second callback does not insert a second `gateway_credit`.
- `Status` other than the documented success value does not call verify and does not credit.
- Verify code for "already verified" does not credit again.
- A bad `sig`, a missing attempt, or an amount mismatch does not credit.
- Reverse within 30 minutes appends a negative ledger row and does not delete the credit. The reverse body is `merchant_id` and `authority` only.
- Explicit gateway rejection rolls the reverse transaction back. A missing gateway response leaves `reverse_pending`.
- Logs contain no merchant id, authority, HMAC, salt, or full card number.
