---
name: confido-legal-accept-a-payment
description: Accept a card or ACH payment from a law-firm client through Confido Legal, either with a hosted Payment Link or with Hosted Fields, routing funds to the firm's operating or trust (IOLTA) account.
api: Confido Legal GraphQL API
endpoint: https://api.gravity-legal.com/
sandbox: https://api.sandbox.gravity-legal.com/
operations:
  - addPaymentLink
  - paymentSessionCreate
  - paymentSessionComplete
  - bankAccountsList
  - transaction
  - recordManualPaymentOnPaymentLink
generated: '2026-09-05'
method: generated
source: graphql/confido-legal-introspection.json + https://docs.confidolegal.com/docs/receive-money/payment-links + https://docs.confidolegal.com/docs/hosted-fields/overview
---

# Accept a payment (Confido Legal)

Confido Legal is a GraphQL-only API. Every request is a `POST` to
`https://api.gravity-legal.com/` (sandbox: `https://api.sandbox.gravity-legal.com/`) carrying
an `x-api-key` header. Use a **Firm token** (`f_secret_*`) for everything on this page.
Apollo CSRF prevention rejects requests that are not preflighted — send
`Content-Type: application/json`.

## Decide which path you need

| Path | Use when | PCI posture |
|---|---|---|
| **Payment Link** (`addPaymentLink`) | Fastest to ship. Confido hosts the page; you send or iframe the URL. | Card data never touches your servers. |
| **Hosted Fields** (`paymentSessionCreate` → `paymentSessionComplete`) | You want full control of the payment UI. | iframes injected by Confido's SDK keep you out of PCI scope. |

## Before you charge: confirm the deposit account

A legal payment usually splits between the firm's **operating** and **trust (IOLTA)**
accounts. Do not assume an `ACTIVE` firm has both.

1. Call the `bankAccountsList` query.
2. Read `accountCategory` on each result — it is either `operating` or `trust`.
3. Keep the `id` of the account each portion of the payment should land in.

If you name a trust split and the firm has no usable default trust account, account
resolution can fail. Validate in your integration rather than assuming.

## Path A — Payment Link

1. `addPaymentLink` with the client, the amount, and the operating/trust split.
2. Give the returned URL to the client, or embed it in an iframe.
3. Watch for the `transaction.created` webhook, then read the transaction with the
   `transaction` query using the id from the webhook payload.
4. If the firm is paid offline instead (check, wire, cash), call
   `recordManualPaymentOnPaymentLink`. The link's outstanding amount updates and its
   status becomes `paid` or `partially_paid`. Manual payments move no money — they are
   recordkeeping only.

## Path B — Hosted Fields

The documented sequence, in order:

1. **Server:** `paymentSessionCreate` → returns a public token (`pay_public_*`).
2. Pass the public token to your frontend. It is one-time-use and safe to expose.
3. Add `div`s with unique `id`s where the fields should appear.
4. Load the SDK: `<script src="https://js.gravity-legal.com/hosted-fields.js">`
   (sandbox: `https://js.sandbox.gravity-legal.com/hosted-fields.js`). Wait for the
   `confido_hosted_fields_script_loaded` `postMessage` before initializing.
5. Initialize `window.confidoHostedFields` with the token. Confido injects the iframes.
6. On submit, call `sdk.submitFields()`. Sensitive data goes straight to Confido.
7. **Server:** `paymentSessionComplete` to finish the payment.

Register every domain that loads the fields under **Settings → Trusted Domains** in the
Partner Portal first. Wildcards like `https://*.myapp.com` are accepted; non-local domains
must be `https`. Sandbox does not enforce trusted domains (but stored *disbursement*
method forms do).

## Reading the result

`Transaction.status_v2` is the field to read — `Transaction.status` is deprecated in
favour of it, and `Transaction.achStatus` is deprecated in favour of `status`. Values:
`PENDING`, `FUNDS_IN_TRANSIT`, `DEPOSITED`, `VOIDED`, `RETURNED`, `REFUNDED`,
`PARTIALLY_REFUNDED`, `CHARGED_BACK`, `HELD`, `SUCCESSFUL`, `ERROR`.

## Rules that bite

- **Rate limit:** 500 requests/minute and 10 concurrent requests per `firmId` (or
  `partnerId` for partner tokens). Over it you get `429`; wait at least 60 seconds, then
  back off exponentially.
- **No idempotency key.** Confido publishes no `Idempotency-Key` header for GraphQL
  mutations. To make a create safely retryable, supply your own UUID v4 in the operation's
  optional client-supplied `id` field where one exists (`disbursementCreate` documents
  this). Otherwise a blind retry can double-charge — read back with `transactionsList`
  before retrying.
- **Errors** come back as GraphQL `errors[]` with HTTP 200 in the normal case. Always
  inspect `errors` even on a 200.

## Testing

Sandbox cards and ACH values are in `sandbox/confido-legal-sandbox.yml`. `4242 4242 4242
4242` succeeds; `4000 3000 1111 2220` fails; any account number with routing `000000000`
fails ACH.
