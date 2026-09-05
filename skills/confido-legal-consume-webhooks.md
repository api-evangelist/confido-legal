---
name: confido-legal-consume-webhooks
description: Register, verify, and reliably process Confido Legal webhooks — HMAC-SHA512 signature checking, eventId idempotency, the retry/dead-letter ladder, and replaying a missed event.
api: Confido Legal GraphQL API
endpoint: https://api.gravity-legal.com/
operations:
  - webhookUrlCreate
  - webhookUrlUpdate
  - webhookUrlDelete
  - webhookUrlsList
  - webhookEventsList
  - webhookEventResend
generated: '2026-09-05'
method: generated
source: graphql/confido-legal-introspection.json + https://docs.confidolegal.com/docs/webhooks/configure + https://docs.confidolegal.com/docs/webhooks/webhook-types
---

# Consume Confido Legal webhooks

## The payload is a pointer, not a record

Confido POSTs an **array** of minimal payloads. Each carries only ids — you then query the
API for current state.

```json
[{
  "data": { "transaction": { "id": "9c2dbhy6-08d9-4d94-971b-8b594c2ea13c" } },
  "type": "transaction.created",
  "firmId": "fcab45p5-c4cb-427d-a144-b3f11f227658",
  "eventId": "2e9e4123-9568-4f34-9ae2-836f760f03b8"
}]
```

## Verify the signature — always

Every request carries an `X-SIGNATURE` header. Compute an **HMAC-SHA512** of the payload
using the webhook secret from the Partner Portal, base64-encode the digest, and compare.

```ts
import crypto from "crypto";
const hmac = crypto.createHmac("sha512", webhookSecret);
hmac.update(JSON.stringify(webhookPayload));
return hmac.digest("base64") === xSignatureHeader;
```

## Make your handler idempotent on `eventId`

`eventId` is the idempotency key. **Resends reuse the same `eventId`**, so a handler that
keys on it is safe against both retries and manual replays. This matters more than usual
here: the Confido GraphQL API itself publishes no `Idempotency-Key` header, so webhook
`eventId` is the one replay-protection primitive the platform hands you.

## The retry ladder

No 2xx within **5 seconds** → the event enters `failed` and is retried at
**5 min → 20 min → 60 min → 1 day → dead letter**.

If an event dead-letters and the URL has had **no successful response in 24 hours**,
Confido **disables the webhook URL**. It must be re-enabled in the Partner Portal. Build an
alert on this — a quiet outage costs you the whole event stream, not just the failures.

## Replay what you missed

- Portal: open the webhook config and resend from the per-URL event list (failed and
  dead-lettered included).
- API, as a Partner Admin: `webhookEventsList` scoped to a `webhookUrlId` to enumerate,
  then `webhookEventResend` with the `eventId`.

There is **no "replay everything since timestamp"**. For a long gap, combine polling the
API with portal resends.

## Register the endpoints

`webhookUrlCreate` / `webhookUrlUpdate` / `webhookUrlDelete` / `webhookUrlsList`, or
**Settings → Webhooks** in the portal. **You receive nothing for types you have not
selected** — a silent stream is usually an unselected type, not a bug.

## The 17 event types

**Transactions** — `transaction.created`, `transaction.funds_in_transit`,
`transaction.deposited`, `transaction.voided`, `transaction.refunded`,
`transaction.partially_refunded`, `transaction.ach_returned`

**Disbursements** — `disbursement.updated` (payload carries both `disbursement` and
`oldDisbursement`, so you can diff the transition)

**Firms** — `firm.updated`

**Statements** — `statement.created`, `statement.updated`

**Stored payment methods** — `stored_payment_method.created`, `.updated`, `.deleted`

**Stored disbursement methods** — `stored_disbursement_method.created`, `.updated`, `.deleted`

## Exercising them in sandbox

`sandboxOnlyMoveTransactionToFundsInTransit`, `sandboxOnlyMoveTransactionToDeposited`,
`sandboxOnlyTriggerAchReturn`, `sandboxOnlyTriggerChargeback`,
`sandboxOnlyTriggerChargebackReversal`, `sandboxOnlyTriggerPrearbitrationLost`,
`sandboxOnlyCreateMockStatement`, `sandboxOnlyUpdateMockStatement`. Each fires the
corresponding webhook. All are rejected in production.
