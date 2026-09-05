---
name: Confido
description: Use when building payment acceptance workflows for legal firms, integrating payment processing APIs, implementing firm onboarding flows, managing client payments and disbursements, or working with Salesforce CRM integrations for legal practice management.
metadata:
    mintlify-proj: confido
    version: "1.0"
---

# Confido Legal Skill

## Product summary

Confido Legal is a GraphQL-based payments platform for legal firms that handles client payments (receive money), disbursements to clients (send money), firm onboarding, and payment method management. The API endpoints are `https://api.sandbox.gravity-legal.com/v2` (sandbox) and `https://api.gravity-legal.com/v2` (production). Agents work with the platform through GraphQL mutations/queries, JavaScript SDKs (onboarding.js and hosted-fields.js), and Salesforce integration packages. All API requests require an `x-api-key` header with the appropriate token type. The GraphQL Playground at https://api.sandbox.gravity-legal.com provides interactive query/mutation documentation.

## When to use

Reach for this skill when:
- Building payment collection flows for legal firms (invoices, retainers, trust deposits)
- Implementing firm onboarding or account connection workflows
- Creating disbursement (payout) flows to send money to clients
- Integrating payment acceptance into web applications via hosted fields or payment links
- Setting up webhooks to track payment status changes
- Working with Salesforce CRM to sync clients, matters, and transactions
- Handling refunds, voids, or payment lifecycle management
- Testing payment flows in sandbox before production deployment

## Quick reference

### API Token Types

| Token Type | Prefix | Use Case | Scope |
|---|---|---|---|
| Partner Token | `p_secret_*` | Create/manage firms | Partner-level operations |
| Firm Token | `f_secret_*` | Most API calls | Scoped to single firm |
| Payment Session Token | `pay_public_*` | Frontend payment forms | One-time payment session |
| Onboarding Token | `onb_public_*` | Firm onboarding form | 24-hour expiry, firm-scoped |

### JavaScript SDK Script Tags

```html
<!-- Onboarding form -->
<script src="https://js.sandbox.gravity-legal.com/onboarding.js" />
<script src="https://js.gravity-legal.com/onboarding.js" /> <!-- production -->

<!-- Payment fields (card/ACH) -->
<script src="https://js.sandbox.gravity-legal.com/hosted-fields.js" />
<script src="https://js.gravity-legal.com/hosted-fields.js" /> <!-- production -->
```

### Key GraphQL Mutations (Firm-level)

| Mutation | Purpose | Returns |
|---|---|---|
| `createOnboardingToken` | Generate 24-hour token for onboarding form | `token` |
| `paymentSessionCreate` | Create token for hosted payment fields | `paymentSessionToken` |
| `createSavePaymentMethodToken` | Create token to save payment method for later | `savePaymentMethodToken` |
| `addPaymentLink` | Create hosted payment page for client | Payment Link object |
| `addClient` | Create or update client record | Client object |
| `addMatter` | Create matter tied to client | Matter object |
| `disbursementCreate` | Create payout to client | Disbursement object |
| `disbursementApprove` | Approve disbursement and generate URL | Disbursement with URL |
| `transactionVoid` | Void a transaction (within cutoff window) | Void request object |
| `transactionRefund` | Refund a transaction | Refund request object |

### Salesforce Configuration

| Setting | Location | Required |
|---|---|---|
| API Token | Setup → Custom Settings → Confido Settings → ApiToken__c | Yes |
| Webhook Secret | Setup → Custom Settings → Confido Settings → WebhookSecret__c | Yes |
| Production Environment | Setup → Custom Settings → Confido Settings → ProductionEnvironment__c | Checkbox |
| Webhook URL | Confido dashboard | `https://<domain>/services/apexrest/confido_core/webhook` |

### Salesforce Objects

- **Client__c** — Legal firm client (syncs to Confido)
- **Matter__c** — Legal matter tied to client
- **PaymentLink__c** — Hosted payment page (amount, status, outstanding balance)
- **Transaction__c** — Auto-created from webhooks (payment, refund, return records)
- **BankAccount__c** — Read-only from Confido (operating/trust accounts)
- **AggregateLink__c** — Combines multiple payment links for full balance payment

## Decision guidance

### When to use Payment Links vs Hosted Fields

| Scenario | Use Payment Links | Use Hosted Fields |
|---|---|---|
| Quick launch, minimal customization | ✓ | |
| Full UI control, custom styling | | ✓ |
| Pre-built payment methods (card, ACH, Apple Pay) | ✓ | |
| Custom form layout or embedded payments | | ✓ |
| Surcharging support needed | ✓ | ✓ |
| Fastest time to market | ✓ | |

### When to use Stored Payment Methods vs Subscriptions

| Scenario | Use Stored Payment Methods | Use Subscriptions |
|---|---|---|
| Manual control over when to charge | ✓ | |
| Automatic recurring charges on schedule | | ✓ |
| Payment plan with custom logic | ✓ | |
| Subscription service with pre-built reminders | | ✓ |
| Multiple charges to same client | ✓ | ✓ |

### When to use Disbursements vs Manual Payments

| Scenario | Use Disbursements | Use Manual Payments |
|---|---|---|
| Send money to client (ACH, card, check) | ✓ | |
| Record offline payment (check, wire, cash) | | ✓ |
| Client must verify identity to accept | ✓ | |
| Instant payout to debit card | ✓ | |
| Recordkeeping only, no funds move | | ✓ |

## Workflow

### Accepting a Payment via Hosted Fields

1. **Backend: Create payment session token** — Call `paymentSessionCreate` mutation with Firm Token, optionally specify `bankAccountId`
2. **Frontend: Include SDK** — Add `<script src="https://js.sandbox.gravity-legal.com/hosted-fields.js" />`
3. **Frontend: Initialize fields** — Call `window.confidoHostedFields.init()` with payment token, specify `activeForm` (card or ach), and container IDs for each field
4. **Frontend: Listen for changes** — Add change listener to monitor field state, enable/disable submit button based on `state.fields` validation
5. **Frontend: Submit fields** — Call `window.confidoHostedFields.submitFields()` when user clicks pay
6. **Backend: Complete payment** — Call `paymentSessionComplete` mutation to finalize transaction
7. **Listen for webhooks** — Receive `transaction.created`, `transaction.deposited`, etc. to track payment status

### Onboarding a New Firm

1. **Backend: Create firm** — Call `createFirm` mutation with Partner Token, get back Firm Token and optional Onboarding Token
2. **Backend: Store Firm Token** — Save in secure database for future API calls
3. **Frontend: Include SDK** — Add `<script src="https://js.sandbox.gravity-legal.com/onboarding.js" />`
4. **Frontend: Render form** — Call `window.confidoOnboarding.renderForm()` with Onboarding Token and container ID
5. **Monitor form events** — Listen for `loaded`, `saved`, `submitted`, `token_expires_soon`, `token_expired` events
6. **Handle token expiry** — If token expires, create new one via `createOnboardingToken` and re-render
7. **Confirm submission** — When `submitted` event fires, firm is sent to underwriting

### Creating a Disbursement (Payout)

1. **Create disbursement** — Call `disbursementCreate` mutation with amount, authorized email/phone, funding account, optional client/matter
2. **Approve disbursement** — Call `disbursementApprove` to move to APPROVED status and generate shareable URL
3. **Distribute URL** — Send URL to client via email or SMS
4. **Client accepts** — Client verifies identity, enters bank details, accepts funds
5. **Track status** — Monitor webhooks for `disbursement.updated`, `transaction.created`, `transaction.funds_in_transit`, `transaction.delivered`
6. **Handle failures** — If ACH returns, webhook sends `transaction.ach_returned` with return reason

### Salesforce: Syncing Payments

1. **Configure settings** — Set API Token, Webhook Secret, Production Environment checkbox in Custom Settings
2. **Expose webhook endpoint** — Create Salesforce Site or Experience Cloud to expose `/webhook` REST resource
3. **Register webhook URL** — In Confido dashboard, point webhook to your Salesforce endpoint
4. **Sync bank accounts** — Run `BankAccountsSync` invocable action to pull operating/trust accounts
5. **Create payment link** — Use `PaymentLinkCreate` invocable action or LWC component
6. **Receive webhooks** — Confido sends webhook → Salesforce verifies HMAC signature → auto-creates Transaction__c records
7. **Query transactions** — Use `confidoTransactionsList` LWC to display recent payments

## Common gotchas

- **Token expiry**: Onboarding Tokens expire after 24 hours. Payment Session Tokens are one-time-use. Always check event listeners for `token_expired` and regenerate.
- **Firm Token security**: Never expose Firm Tokens to frontend. Keep them server-side only. Payment Session and Onboarding Tokens are safe to share with clients.
- **Void/refund timing**: Voids only work within a cutoff window (11 PM CT for cards, varies for ACH). Check `transaction.canVoid` boolean before attempting. Refunds and voids now support async processing — may return `AWAITING_RESULT` status; listen for webhooks to confirm completion.
- **Aggregate links don't support partial payments**: If using Aggregate Payment Links, client must pay full balance. Use individual Payment Links if partial payments are needed.
- **Surcharging on voids/refunds**: Surcharge amounts are automatically voided/refunded proportionally. Don't manually adjust.
- **ACH payment limits**: Check `state.paymentLink.maxAchPayment` in hosted fields state to warn users of limits.
- **Webhook HMAC verification**: Always verify `X-SIGNATURE` header using HMAC-SHA512 with your Webhook Secret. Unverified webhooks are a security risk.
- **Salesforce sync errors**: Check `SyncErrorMessage__c` field on Client__c, Matter__c, PaymentLink__c, AggregateLink__c if sync fails. Errors clear on next successful sync.
- **Bank account management**: Bank accounts are read-only in Salesforce and managed in Confido. Sync them regularly with `BankAccountsSync` action.
- **Hosted fields state**: Always check `state.initializing` and `state.isSubmitting` before allowing user interaction. Don't submit while fields are loading.
- **Card type detection**: `bin_loaded` event fires after BIN lookup completes. Use this to show card brand and determine if surcharging applies.
- **Payment method switching**: Call `setActiveForm('card')` or `setActiveForm('ach')` to switch between card and ACH fields. Ensure DOM containers exist before switching.

## Verification checklist

Before submitting work with Confido:

- [ ] API token is correct type (Partner, Firm, Payment Session, or Onboarding) for the operation
- [ ] `x-api-key` header is included in all API requests
- [ ] Sandbox vs production endpoints match the environment (sandbox.gravity-legal.com vs gravity-legal.com)
- [ ] Onboarding Token is created and passed to frontend before rendering form
- [ ] Payment Session Token is created before initializing hosted fields
- [ ] All required field containers exist in DOM before calling `init()` or `setActiveForm()`
- [ ] Change event listeners are attached to monitor field state and form progress
- [ ] Webhook endpoint is registered in Confido dashboard and exposed via Salesforce Site/Community
- [ ] Webhook Secret is stored in Salesforce Custom Settings and matches Confido configuration
- [ ] HMAC signature verification is implemented for incoming webhooks
- [ ] Test payment methods are used in sandbox (e.g., 4242 4242 4242 4242 for success)
- [ ] Void/refund cutoff times are checked before attempting (11 PM CT for cards)
- [ ] Async refund/void handling includes listening for `AWAITING_RESULT` status and webhooks
- [ ] Aggregate links are only used when full balance payment is required (no partial payments)
- [ ] Bank accounts are synced before creating payment links in Salesforce
- [ ] Transaction records are auto-created from webhooks (don't manually create)
- [ ] Error messages from sync failures are logged and reviewed

## Resources

- **Comprehensive page listing**: https://docs.confidolegal.com/llms.txt
- **GraphQL Playground** (interactive queries/mutations): https://api.sandbox.gravity-legal.com/v2
- **Onboarding.js SDK Reference**: https://docs.confidolegal.com/docs/onboarding-js/overview
- **Hosted Fields SDK Reference**: https://docs.confidolegal.com/docs/hosted-fields-js/overview
- **Salesforce Integration Guide**: https://docs.confidolegal.com/docs/salesforce/overview

---

> For additional documentation and navigation, see: https://docs.confidolegal.com/llms.txt