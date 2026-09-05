---
name: confido-legal-reverse-a-payment
description: Take back a Confido Legal payment — void it before it settles, refund it (fully or partially) after it settles, and handle the asynchronous AWAITING_RESULT path and ACH returns.
api: Confido Legal GraphQL API
endpoint: https://api.gravity-legal.com/
operations:
  - transactionVoidDetails
  - transactionVoid
  - transactionRefund
  - transactionVoidOrRefund
  - voidRequestGet
  - refundRequestGet
  - transaction
  - storedPaymentMethodRefund_v2
generated: '2026-09-05'
method: generated
source: graphql/confido-legal-introspection.json + https://docs.confidolegal.com/docs/payments-lifecycle/voids-and-refunds + https://docs.confidolegal.com/docs/payments-lifecycle/ach-returns
---

# Reverse a payment (Confido Legal)

Before you reverse anything, read the two booleans Confido puts on the `Transaction`
object: **`canVoid`** and **`canRefund`**. Both can be `false` at once — that happens while
the money is in transit.

## Void — before settlement

Voiding pulls the transaction out of the batch; no funds move.

The window is a **same-day cutoff in Central Time**, set by firm settings and payment method:

| Payment method | Cutoff (Central) | Legacy cutoff (Central) |
|---|---|---|
| Card | 11:00 PM | 10:30 PM |
| ACH | 11:00 PM | 12:30 PM & 9:00 PM |

**Always call `transactionVoidDetails` first.** A single payment can produce several
transactions, and voiding requires cancelling all of them together —
`transactionVoidDetails` tells you which ones are affected. Then call `transactionVoid`.

Payments taken on an **Aggregate Payment Link cannot be voided.**

## Refund — after settlement

`transactionRefund` works at any time once the transaction has settled. It takes an
optional `amount` for a partial refund. Refunding more than the original amount throws.

For a stored payment method, use `storedPaymentMethodRefund_v2` — `storedPaymentMethodRefund`
is deprecated.

## Either/or

`transactionVoidOrRefund` picks the right one for you. It does **not** support partial
amounts — if you need a partial, choose the specific mutation.

## The asynchronous path — do not treat a response as a result

Confido waits up to **30 seconds** for the financial institution:

1. Completes inside 30s → you get the original and the new refund/void transactions back.
2. Still processing → you get a request in status **`AWAITING_RESULT`**.
3. Poll `refundRequestGet` / `voidRequestGet`, or read the `refundRequests` and
   `voidRequests` fields on the `transaction` query, or wait for the
   `transaction.refunded` / `transaction.partially_refunded` / `transaction.voided` webhook.

`Transaction.status_v2` reflects the final outcome. Do not read the deprecated
`Transaction.status`.

## Surcharges are handled for you

- **Void:** any surcharge on the original transaction is voided with it.
- **Refund:** surcharge is refunded proportionally to the refund amount.

## ACH returns are not reversals you initiate

An already-settled ACH payment can be **returned** by the bank, reversing funds that may
already have reached the firm's account. Partial returns do not exist. Subscribe to
`transaction.ach_returned`. To exercise the path in sandbox, call
`sandboxOnlyTriggerAchReturn` on a transaction with `type: achPayment` — it simulates an
Insufficient Funds return and fires the webhook.

## Deprecated names to avoid

`refundTransaction`, `voidTransaction` and `voidOrRefundTransaction` are all deprecated.
Use `transactionRefund`, `transactionVoid` and `transactionVoidOrRefund`.
