---
name: confido-legal-send-a-disbursement
description: Send settlement or vendor funds out of a law firm's account through Confido Legal — create, approve, track and void a disbursement, singly or in bulk.
api: Confido Legal GraphQL API
endpoint: https://api.gravity-legal.com/
sandbox: https://api.sandbox.gravity-legal.com/
operations:
  - disbursementCreate
  - disbursementsCreateBulk
  - disbursementApprove
  - disbursementUpdate
  - disbursementReturnToDraft
  - disbursementVoid
  - disbursementDelete
  - disbursementGet
  - disbursementsList
  - disbursementEventsGet
  - bankAccountsList
generated: '2026-09-05'
method: generated
source: graphql/confido-legal-introspection.json + https://docs.confidolegal.com/docs/send-money/create-disbursement + https://docs.confidolegal.com/docs/send-money/bulk-disbursements
---

# Send a disbursement (Confido Legal)

Authenticate with a **Firm token** (`f_secret_*`) in the `x-api-key` header against
`https://api.gravity-legal.com/`.

## Create

`disbursementCreate` takes a `DisbursementCreateInput!`. Required:

- **`amount`** — integer, in **cents**. Must be `> 0` and `<= 100000000` ($1,000,000.00).
- **`authorizedIdentities`** — at least one. Each is `{ type: EMAIL | PHONE, value: String }`,
  with an optional informational `source` (`CLIENT`, `PRIMARY_CONTACT`, `CONTACT`, `MANUAL`).
  These are the identities allowed to access and accept the disbursement.
- **`fundingAccountId`** — the bank account the money is pulled from. Get it from
  `bankAccountsList`.

Useful optional inputs:

- **`id`** — a client-supplied UUID v4. **Set this.** It is the only replay protection
  Confido offers on this mutation: reusing your own id lets you retry a create without
  a second payout.
- **`status`** — `DRAFT` (the default) or `APPROVED`. Only `FIRM_ADMIN` users may create
  directly in `APPROVED`.
- **`allowedMethods`** — restrict from the firm's defaults. Values: `ACH`, `ACH_DIRECT`,
  `ACH_PLAID`, `PAYPAL`, `PUSH_TO_CARD`, `REQUEST_CHECK`, `ZELLE`.

**A disbursement only gets a `url` once it reaches `APPROVED`.** A `DRAFT` has no link
to send.

## Approve, amend, withdraw

- `disbursementApprove` moves `DRAFT` → `APPROVED` and mints the recipient URL.
- `disbursementReturnToDraft` pulls an approved disbursement back. (`disbursementUnapprove`
  still exists but is deprecated in favour of it.)
- `disbursementUpdate` amends a disbursement.
- `disbursementVoid` cancels one.
- `disbursementDelete` removes one.

## Bulk

`disbursementsCreateBulk` runs many creates in one request and returns results
synchronously. It still counts against the same per-firm limits and can return `429` if
the firm is already near the cap. Read `displayNumber` on each result item —
`disbursementNumber` is deprecated.

## Track

- `disbursementGet` / `disbursementsList` for state.
- `disbursementEventsGet` for the audit trail: `CREATED`, `UPDATED`, `APPROVED`,
  `UNAPPROVED`, `IDENTITY_ENTERED`, `LOGIN_ATTEMPT`, `MFA_ENTERED`, `ACCEPTED`,
  `DELIVERED`, `FAILED`, `EXPIRED`, `VOID`.
- Statuses: `DRAFT`, `REQUEST_PENDING`, `APPROVED`, `CHECK_REQUESTED`, `FUNDS_IN_TRANSIT`,
  `DELIVERED`, `EXPIRED`, `FAILED`, `VOID`.
- Subscribe to the `disbursement.updated` webhook. The payload carries both
  `disbursement` and `oldDisbursement`, so you can diff the status transition.
- Read `depositEta`, not `expectedDeliveryDate` (deprecated).

## Cost, so you can tell the firm

Confido's published pricing is **$3 per disbursement**. The recipient chooses standard
ACH at no cost (4–6 days) or pays **1%, capped at $95** for faster delivery.

## Testing

`sandboxOnlyDisbursementExpire` forces the expiry path in sandbox. These `sandboxOnly*`
mutations are rejected in production.
