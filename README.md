# Open Banking Architecture Patterns

Production-minded reference patterns for engineers integrating Open Banking, payment initiation and account-information APIs.

This repository turns complex payment flows into implementation-ready guidance: sequence diagrams, trust boundaries, failure modes, webhook controls and operational checklists. It is designed for platform teams, solution architects, technical writers and developer-experience teams who need documentation that closes the gap between an API reference and a resilient production integration.

> **Scope:** The examples are provider-neutral. Placeholder headers such as `Provider-Signature` represent controls whose exact names and formats vary by platform. Always confirm endpoints, event names, status values and cryptographic requirements against your provider's current documentation.

## What this repository demonstrates

- End-to-end Payment Initiation Service (PIS) flows, including decoupled authorisation
- Account Information Service (AIS) consent and data-access patterns
- Signed API requests, idempotency and safe retry behaviour
- Authenticated webhook handling with replay and duplicate protection
- Explicit separation of provider state, internal state and settlement state
- Documentation patterns that reduce integration ambiguity and support burden

## Reference architecture: payment initiation

The following flow uses a detached JWS request signature as a concrete example. It deliberately keeps the internal payment state separate from the provider's external status vocabulary.

```mermaid
sequenceDiagram
    autonumber
    actor U as User
    participant C as Client App
    participant B as Client Backend
    participant G as Provider API Gateway
    participant A as Bank / ASPSP
    participant W as Webhook Endpoint
    participant D as Payment Database

    U->>C: Confirm payment
    C->>B: POST /payments (order_id)
    B->>D: Reserve idempotency key
    B->>B: Sign method + path + headers + body
    Note over B,G: Example: Provider-Signature contains a JWS and key ID
    B->>G: Create payment + Provider-Signature + Idempotency-Key
    G->>G: Verify signature and validate request
    G-->>B: Payment ID + authorisation action
    B->>D: Store provider ID; state = PENDING_AUTHORISATION
    B-->>C: Return redirect / authorisation URI
    C->>A: Redirect user for bank authorisation
    U->>A: Authenticate and approve
    A-->>C: Redirect to client return URI
    C->>B: Resume payment-status view
    B->>G: GET payment status
    G-->>B: Current provider status
    B->>D: Reconcile internal state

    par Asynchronous lifecycle update
        G->>W: Signed webhook + event ID
        W->>W: Preserve raw body; verify JWS and trusted key
        W->>W: Check replay window and deduplicate event ID
        W->>D: Atomically apply valid state transition
        D-->>W: Commit result
        W-->>G: 2xx acknowledgement
    and Client polling or refresh
        C->>B: GET /payments/{order_id}
        B->>D: Read internal payment state
        B-->>C: PENDING / EXECUTED / SETTLED / FAILED
    end
```

### Why the flow is designed this way

1. **Signing happens on the backend.** Private signing keys never belong in browser or mobile code.
2. **Creation is idempotent.** A stable key prevents a timeout or client retry from creating another payment.
3. **Redirect completion is not payment completion.** The return URI resumes the user journey; it is not trusted as proof of settlement.
4. **Webhook bytes are preserved.** Signature verification must use the exact request material required by the provider—not a reserialised approximation.
5. **Authentication precedes mutation.** No database state changes occur before signature, freshness and duplicate checks pass.
6. **Updates are atomic and monotonic.** Duplicate or late events cannot regress a terminal payment state.
7. **Polling and webhooks reconcile.** The webhook is the normal asynchronous path; a status lookup supports recovery when delivery is delayed or exhausted.

## Status mapping: keep three meanings separate

`PENDING → EXECUTED → SETTLED` is a useful **internal abstraction**, but it is not a universal Open Banking status model.

| Internal state | Meaning in this reference | Illustrative UK Open Banking mapping* |
| --- | --- | --- |
| `PENDING` | Authorisation or processing is incomplete | `Pending` |
| `EXECUTED` | Accepted for execution / settlement is in process | `AcceptedSettlementInProcess` |
| `SETTLED` | Settlement on the debtor account is complete | `AcceptedSettlementCompleted` |
| `FAILED` | Rejected, failed or irrecoverably expired | `Rejected` or provider-specific terminal failure |

\*Mappings are illustrative. A production adapter must define provider- and scheme-specific transitions explicitly.

## Webhook acceptance algorithm

```text
receive(request):
  raw_body = request.raw_body
  signature = request.headers[provider_signature_header]

  if !verify(signature, method, path, headers, raw_body, trusted_key):
      reject 401

  if timestamp_outside_allowed_window(request):
      reject 401

  begin transaction
    if event_id_already_processed(request.event_id):
        commit
        return 204

    payment = lock_payment(request.payment_id)
    assert transition_is_allowed(payment.state, request.status)
    update payment and record event_id
  commit

  return 204
```

The implementation should return a fast success response only after durable acceptance. Slow downstream work—notifications, ledger enrichment or analytics—should run from an internal queue or outbox.

## Repository map

```text
open-banking-architecture-patterns/
├── README.md
├── pis/
│   ├── payment-initiation-flow.md
│   ├── request-signing.md
│   ├── idempotency-and-retries.md
│   └── status-mapping.md
├── ais/
│   ├── consent-lifecycle.md
│   ├── data-access-flow.md
│   └── reauthorisation-and-revocation.md
├── developer-experience/
│   ├── integration-quickstart.md
│   ├── error-taxonomy.md
│   └── production-readiness-checklist.md
├── webhook-resilience/
│   ├── signature-verification.md
│   ├── replay-and-deduplication.md
│   ├── retry-state-machine.md
│   └── reconciliation.md
├── examples/
│   ├── node/
│   └── python/
└── assets/
    └── diagrams/
```

## Production-readiness questions

Before launch, the integration team should be able to answer:

- Where are signing keys stored, rotated and audited?
- Which exact method, path, headers and body bytes are covered by the signature?
- What makes a payment-creation retry safe?
- Which provider statuses map to each internal state?
- Which state transitions are terminal, and can late events regress them?
- How are webhook signatures, key identifiers and timestamps validated?
- How long are event IDs retained for deduplication?
- What happens after the provider exhausts webhook retries?
- Can operations replay an event safely without duplicating side effects?
- Which metrics reveal signature failures, retry storms and reconciliation drift?

## Planned patterns

- **PIS:** [Payment Initiation Integration Guide](pis/payment-initiation-flow.md), followed by a failure-state catalogue and refund lifecycle
- **AIS:** consent lifecycle, pagination, rate limits and data freshness
- **Webhook resilience:** key rotation, queue-backed ingestion and reconciliation
- **Developer experience:** runnable verification examples, test fixtures and troubleshooting decision trees

## Sources and standards

- [Open Banking UK: Domestic Payments v3.1.11](https://openbankinguk.github.io/read-write-api-site3/v3.1.11/resources-and-data-models/pisp/domestic-payments.html)
- [RFC 7515: JSON Web Signature](https://www.rfc-editor.org/rfc/rfc7515)
- [RFC 9110: HTTP Semantics](https://www.rfc-editor.org/rfc/rfc9110)

## About this work

This is an independent architecture and documentation project. It is not affiliated with or endorsed by any API provider, financial institution or standards body.

I help fintech product and platform teams turn complex integrations into clear, testable developer journeys through architecture diagrams, production-ready integration guides and webhook-resilience blueprints.

**Available engagement:** a focused two-week Developer Experience sprint covering discovery, documentation architecture, diagrams, implementation guidance and editorial handover.

For enquiries, use [SyncYourCloud](https://www.syncyourcloud.io/).

## Licence

Documentation is released under the [Creative Commons Attribution 4.0 International licence](https://creativecommons.org/licenses/by/4.0/). Any accompanying source-code examples should be released under the MIT licence.
