# Documentation by user need

The repository uses [Diátaxis](https://diataxis.fr/) to separate learning, task completion, lookup and conceptual understanding. Choose the section that matches what you are trying to do.

## Tutorial — learn by doing

- [Build and test a local webhook receiver](../tutorials/local-webhook-lab.md)

The tutorial provides a safe, provider-neutral exercise. It is deliberately simpler than a production implementation.

## How-to guides — complete a task

- [Initiate a payment safely](../pis/payment-initiation-flow.md)
- [Sign an outbound API request](../pis/request-signing.md)
- [Make payment creation retry-safe](../pis/idempotency-and-retries.md)
- [Implement an account-data access flow](../ais/data-access-flow.md)
- [Verify and process a signed payment webhook](../webhook-resilience/verify-signed-payment-webhook.md)
- [Reconcile delayed or missing payment updates](../webhook-resilience/reconciliation.md)

## Reference — look up rules and decisions

- [Payment status mapping](../pis/status-mapping.md)
- [Consent expiry, reauthorisation and revocation](../ais/reauthorisation-and-revocation.md)
- [Error taxonomy](../developer-experience/error-taxonomy.md)
- [Webhook retry state machine](../webhook-resilience/retry-state-machine.md)
- [Production-readiness checklist](../developer-experience/production-readiness-checklist.md)

## Explanation — understand the design

- [The account-information consent lifecycle](../ais/consent-lifecycle.md)
- [Replay, duplicate delivery and event ordering](../webhook-resilience/replay-and-deduplication.md)

## Scope and evidence

These patterns are provider-neutral. Names such as `Provider-Signature`, example status values and endpoint paths are placeholders. A production integration must use the selected provider's current API, security profile, consent model and operational contract.

Standards used throughout the repository:

- [Open Banking UK Read/Write API v4.0.1](https://openbankinguk.github.io/read-write-api-site3/v4.0.1/)
- [RFC 7515 — JSON Web Signature](https://www.rfc-editor.org/rfc/rfc7515)
- [RFC 7517 — JSON Web Key](https://www.rfc-editor.org/rfc/rfc7517)
- [RFC 9110 — HTTP Semantics](https://www.rfc-editor.org/rfc/rfc9110)
