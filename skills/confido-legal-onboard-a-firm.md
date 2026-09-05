---
name: confido-legal-onboard-a-firm
description: Get a law firm live on Confido Legal from a partner application — create or connect the firm, mint its API token, embed the onboarding form, and track it to ACTIVE.
api: Confido Legal GraphQL API
endpoint: https://api.gravity-legal.com/
operations:
  - createFirm
  - createFirmSignUpLink
  - firmApiTokenExchangeCode
  - firmApiTokenCreate
  - firmApiTokenList
  - firmApiTokenDelete
  - createOnboardingToken
  - firm
  - firmsList
  - firmUpdate
  - disconnectFromPartner
generated: '2026-09-05'
method: generated
source: graphql/confido-legal-introspection.json + https://docs.confidolegal.com/docs/firm-onboarding/connect + https://docs.confidolegal.com/docs/firm-onboarding/firm-status + https://docs.confidolegal.com/onboarding-js/overview
---

# Onboard a firm (Confido Legal)

In Confido's model a **Partner** is a software company; a **Firm** is a law firm under it.
Everything here starts from a **Partner token** (`p_secret_*`) and ends with a **Firm
token** (`f_secret_*`) you store per firm.

Get a partner account by emailing Confido — the Partner Portal issues partner tokens under
**Settings**.

## Three ways in

**1. The firm already has a Confido account → Connect.**
Configure your logo, Connect URL and Callback URL under **Settings → Connect** in the
Partner Portal. Send the user to your unique Connect URL (an optional `state` query
parameter is threaded back to you unchanged). On authorization Confido redirects to
`https://your_app.com/callback?code=connect_code&state=your_state`. Exchange the
one-time `code` with **`firmApiTokenExchangeCode`** using your Partner token.
(`exchangeCodeForFirmApiToken` is the deprecated name.)

**2. The firm has no account, and you want them in your UI → Onboarding.js.**
- Call `createOnboardingToken` with a Firm token → an `onboarding_public_*` token, short-lived.
- Load `<script src="https://js.gravity-legal.com/onboarding.js">` (sandbox:
  `https://js.sandbox.gravity-legal.com/onboarding.js`).
- Call `window.confidoOnboarding.renderForm` with the token and a `containerId`.
- The form talks to Confido directly; the user never leaves your app.
- Owners holding **more than 25%** of the business must supply personal information —
  use owner invite links for owners who are not the person at the keyboard.

**3. You want Confido to host the whole signup → Sign Up Links.**
`createFirmSignUpLink` returns a link you hand off.

You can also create the firm outright with **`createFirm`**, which can mint a Firm token
in the same call.

## Track the firm to ACTIVE

`firm.status` walks these values, and a **`firm.updated` webhook fires on every change**:

| Status | Meaning |
|---|---|
| `CREATED` | Firm exists; application untouched. |
| `APP_IN_DRAFT` | Application being filled out. |
| `APP_SUBMITTED` | Submitted, waiting for the authorized signer to sign. |
| `APP_IN_REVIEW` | Signed, under underwriting review. |
| `ACTIVE` | Passed underwriting; can process payments. |
| `DECLINED` | Rejected. |
| `HELD` / `SUSPENDED` | Processing paused. |
| `INACTIVE` | No longer able to process payments. |

Do not attempt to charge before `ACTIVE`.

## Managing firm tokens

- A firm can hold **several** tokens — `firmApiTokenCreate` mints another,
  `firmApiTokenList` enumerates them, `firmApiTokenDelete` revokes one.
  (`createFirmApiToken` is deprecated.)
- Tokens are **long-lived**: no TTL, no refresh, no rotate endpoint. Rotate by creating a
  new token and revoking the old one under **Settings → API Tokens**. Revocation takes
  effect immediately.
- Firm tokens are **not scoped** — there is no read-only variant. Any valid firm token can
  do anything the product allows server-side. Keep them on your server, never in a browser.
- **No webhook fires** when a token is created or revoked. Handle `401`/`403` by checking
  the portal.
- Sandbox and production tokens are distinct; the environment is encoded in the prefix
  (`p_secret_sandbox_…`).

`disconnectFromPartner` ends the relationship.

## Testing the whole ladder in sandbox

`createFirm` accepts `mockOnboarding`, and these sandbox-only mutations drive the states:
`sandboxOnlyFillOnboardingData`, `sandboxOnlySubmitOnboardingData`,
`sandboxOnlySendFirmToReview`, `sandboxOnlyActivateFirm`,
`sandboxOnlyFirmSurchargingUpdate`. All are rejected in production.
