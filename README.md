# Open Banking Architecture Patterns

**Design payment and account-data integrations that remain correct when requests time out, customers abandon authorisation, and webhooks arrive late, twice or out of order.**

API references explain which endpoint to call. This repository documents what engineering teams still need to decide around those calls: trust boundaries, internal state, idempotency, consent, signed webhooks, reconciliation and operational evidence.

It is a provider-neutral, production-minded documentation portfolio for backend engineers, solution architects and developer-experience teams working with Open Banking, payment initiation and account-information APIs.

> **Scope:** Placeholder paths, headers and statuses illustrate patterns—not a provider contract. Confirm every endpoint, event, status, algorithm and cryptographic input against the provider and scheme version you implement.

## Start here

| If you need to… | Start with |
| --- | --- |
| Understand the complete payment journey | [Payment Initiation Integration Guide](pis/payment-initiation-flow.md) |
| Prevent duplicate payments during uncertain retries | [Idempotency and safe retries](pis/idempotency-and-retries.md) |
| Authenticate asynchronous payment events | [Signed webhook verification](webhook-resilience/verify-signed-payment-webhook.md) |
| Implement account-data consent and retrieval | [Account-data access flow](ais/data-access-flow.md) |
| Learn the webhook controls through an exercise | [Local webhook tutorial](tutorials/local-webhook-lab.md) |
| Assess an integration before launch | [Production-readiness checklist](developer-experience/production-readiness-checklist.md) |
| Browse by documentation purpose | [Diátaxis documentation map](docs/README.md) |

## What this repository demonstrates

- An end-to-end Payment Initiation Service flow with decoupled user authorisation
- Explicit separation of provider status, internal payment state and settlement meaning
- Backend request signing without exposing private keys to client applications
- Stable idempotency across connection failures, timeouts and safe retries
- Authenticated webhook processing with freshness, deduplication and transition controls
- Account Information Service consent, data-access and revocation patterns
- Recovery through bounded reconciliation when notifications are delayed or exhausted
- Documentation structured with Diátaxis so learning, tasks, lookup and explanation stay distinct

## Documentation model

This repository uses the [Diátaxis](https://diataxis.fr/) framework while retaining domain-based navigation for payments teams. Every document states its content type and serves one primary user need:

| Type | User need | Repository example |
| --- | --- | --- |
| Tutorial | Learn by completing a guided exercise | [Build and test a local webhook receiver](tutorials/local-webhook-lab.md) |
| How-to guide | Complete a defined implementation task | [Initiate a payment safely](pis/payment-initiation-flow.md) |
| Reference | Look up precise technical information | [Payment status mapping](pis/status-mapping.md) |
| Explanation | Understand a system or design decision | [Replay, duplicate delivery and event ordering](webhook-resilience/replay-and-deduplication.md) |

Separating these purposes keeps task instructions concise while allowing deeper concepts and exhaustive technical fields to live in dedicated documents.

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

## Documentation paths

Start with [Documentation by user need](docs/README.md). It routes readers by Diátaxis purpose instead of mixing learning, tasks, lookup material and conceptual background.

### Tutorial

- [Build and test a local webhook receiver](tutorials/local-webhook-lab.md)

### How-to guides

- [Payment initiation](pis/payment-initiation-flow.md)
- [Request signing](pis/request-signing.md)
- [Idempotency and retries](pis/idempotency-and-retries.md)
- [Account-data access](ais/data-access-flow.md)
- [Signed webhook verification](webhook-resilience/verify-signed-payment-webhook.md)
- [Payment reconciliation](webhook-resilience/reconciliation.md)

### Reference

- [Payment status mapping](pis/status-mapping.md)
- [Consent expiry, reauthorisation and revocation](ais/reauthorisation-and-revocation.md)
- [Error taxonomy](developer-experience/error-taxonomy.md)
- [Webhook retry state machine](webhook-resilience/retry-state-machine.md)
- [Production-readiness checklist](developer-experience/production-readiness-checklist.md)

### Explanation

- [Account-information consent lifecycle](ais/consent-lifecycle.md)
- [Replay, duplicate delivery and event ordering](webhook-resilience/replay-and-deduplication.md)

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

## Sources and standards

- [Open Banking UK: Read/Write API v4.0.1](https://openbankinguk.github.io/read-write-api-site3/v4.0.1/)
- [RFC 7515: JSON Web Signature](https://www.rfc-editor.org/rfc/rfc7515)
- [RFC 9110: HTTP Semantics](https://www.rfc-editor.org/rfc/rfc9110)

## About this work

This is an independent architecture and documentation project. It is not affiliated with or endorsed by any API provider, financial institution or standards body.

I help fintech product and platform teams turn complex integrations into clear, testable developer journeys through architecture diagrams, production-ready integration guides and webhook-resilience blueprints.

**Available engagement:** a focused two-week Developer Experience sprint covering discovery, documentation architecture, diagrams, implementation guidance and editorial handover.

For enquiries, use [SyncYourCloud](https://www.syncyourcloud.io/).

## Licence

Documentation is released under the [Creative Commons Attribution 4.0 International licence](https://creativecommons.org/licenses/by/4.0/). Any accompanying source-code examples should be released under the MIT licence.
