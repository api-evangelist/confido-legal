---
generated: '2026-09-05'
method: probed
source: https://api.gravity-legal.com/ (live GraphQL introspection, 2026-09-05, HTTP 200, unauthenticated)
---

# Confido Legal GraphQL API

The Confido Legal GraphQL API is the unified developer interface for the Confido payments
platform. Partners and law-firm developers use it to tokenize payment methods, accept ACH
and card payments with automated routing to operating or trust (IOLTA) accounts, issue
real-time disbursements, manage payors and payees, and reconcile transactions. The API
enforces legal-industry compliance rules (PCI scope reduction, IOLTA segregation, state
surcharge rules) at the platform layer so partners do not have to replicate them.

**GraphQL is the only machine-readable contract Confido Legal publishes.** There is no
OpenAPI, no gRPC, no WSDL, and no AsyncAPI — every one of those was probed and is absent.
The schema is therefore the whole integration surface.

## Endpoints

| Environment | Endpoint |
|---|---|
| Production | https://api.gravity-legal.com/ |
| Sandbox | https://api.sandbox.gravity-legal.com/ |
| Playground | https://studio.apollographql.com/graph/Confido-Legal-vqze3p/variant/current/explorer |

**Documentation:** https://docs.confidolegal.com/

## Schema

| Measure | Count |
|---|---|
| Types | 285 |
| Object types | 155 |
| Enums | 26 |
| Query fields | 41 |
| Mutation fields | 98 |
| Subscription fields | 0 (`subscriptionType` is null) |
| Deprecated members | 26, each naming its replacement |

## Files in this directory

| File | What it is |
|---|---|
| `confido-legal.graphql` | The SDL, rendered from the live introspection document. |
| `confido-legal-introspection.json` | The raw introspection response, verbatim. |

## Calling it

Every request is a `POST` carrying `Content-Type: application/json` and an `x-api-key`
header. Apollo Server CSRF prevention is enabled, so a request that is not a preflighted
GraphQL POST is refused with HTTP 400 and the message *"This operation has been blocked as
a potential Cross-Site Request Forgery (CSRF)"* — this is why every `/.well-known/` probe
against the API host answers 400 rather than 404.

Introspection itself needs no credential.

```sh
curl -X POST https://api.gravity-legal.com/ \
  -H "Content-Type: application/json" \
  -H "apollo-require-preflight: true" \
  -d '{"query":"{ __schema { queryType { name } } }"}'
```

## Errors

Errors arrive in the GraphQL `errors[]` array and can accompany HTTP 200 — always inspect
`errors`, never the status alone. Every observed failure returns
`extensions.code: INTERNAL_SERVER_ERROR` regardless of cause, so the machine-readable code
carries no information. See `errors/confido-legal-problem-types.yml`.

## Related artifacts

- `authentication/confido-legal-authentication.yml` — the four token types
- `conventions/confido-legal-conventions.yml` — idempotency, reversibility, pagination
- `data-model/confido-legal-data-model.yml` — the entity graph
- `lifecycle/confido-legal-lifecycle.yml` — all 26 deprecations
- `skills/` — packaged agent skills grounded in real operationIds
