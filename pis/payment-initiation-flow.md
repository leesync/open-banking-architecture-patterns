# Payment Initiation Integration Guide

This guide explains how to design a provider-neutral payment initiation flow that remains reliable across user redirects, delayed status updates, duplicate requests and webhook retries.

It is intended for backend engineers, solution architects and implementation teams integrating a regulated payment API. Endpoint names and status values are illustrative: replace them with the contract published by your chosen provider.

## What you will build

By the end of this guide, your integration will be able to:

- create a payment without duplicating it during a retry;
- send signed requests from a trusted backend;
- hand the user to their bank for authorisation;
- resume the customer journey without treating a redirect as proof of payment;
- authenticate, deduplicate and process asynchronous webhooks; and
- reconcile provider status with an internal payment record.

## Before you begin

You need:

- access to the provider's sandbox environment;
- a backend service capable of keeping credentials and signing keys secret;
- the provider's current API and webhook specifications;
- a public HTTPS webhook endpoint;
- durable storage for payments, idempotency records and processed event IDs; and
- test credentials for at least one simulated bank or authorisation journey.

Do not place private keys, client secrets or backend access tokens in browser or mobile application code.

## Conceptual endpoints

This guide uses the following placeholder interface:

| Operation | Illustrative endpoint | Purpose |
| --- | --- | --- |
| Create payment | `POST /payments` | Creates a payment resource and returns an authorisation action |
| Read payment | `GET /payments/{payment_id}` | Retrieves the provider's current payment state |
| Receive event | `POST /webhooks/payments` | Receives asynchronous lifecycle notifications |
| Read internal state | `GET /orders/{order_id}/payment` | Returns the state your application exposes to its client |

Your provider may use different resources, versions or authorisation operations. Treat its published API contract as authoritative.

## 1. Define the internal payment state

Do not copy provider statuses directly into application behaviour. Define a small internal state model and maintain an explicit mapping for every provider you integrate.

| Internal state | Meaning | Example customer message |
| --- | --- | --- |
| `CREATED` | The order exists, but no provider payment has been created | Ready to pay |
| `PENDING_AUTHORISATION` | The payment exists and requires user action | Approve the payment with your bank |
| `PROCESSING` | Authorisation succeeded, but the outcome is not final | Your payment is processing |
| `SUCCEEDED` | The integration has received the configured success signal | Payment complete |
| `FAILED` | A terminal failure or rejection occurred | Payment unsuccessful |
| `CANCELLED` | The user or an authorised system cancelled the payment | Payment cancelled |

Before implementation, document:

- every provider-to-internal status mapping;
- which states are terminal;
- which transitions are permitted; and
- which signal your business treats as success—for example, acceptance for execution or completed settlement.

A response indicating that a payment was accepted for processing does not necessarily mean settlement has completed.

## 2. Generate an idempotency key

Create the idempotency key on your backend before calling the provider. Associate it with the business operation, not an individual network attempt.

```text
idempotency_key = "payment:" + order_id + ":v1"
```

Store the key and its intended request fingerprint before sending the request. Reuse the same key when retrying the same logical operation. Generate a new key only when the customer intentionally starts a new payment.

The request fingerprint can include stable fields such as:

- order identifier;
- amount in minor units;
- currency;
- beneficiary identifier; and
- payment type.

If a reused key arrives with different material fields, reject the operation locally rather than sending an ambiguous request.

## 3. Sign and send the creation request

Build the request exactly as required by the provider. A signing scheme may cover the HTTP method, request path, selected headers and exact body bytes.

```http
POST /payments HTTP/1.1
Authorization: Bearer <backend-access-token>
Content-Type: application/json
Idempotency-Key: payment:order-8472:v1
Provider-Signature: <detached-JWS-or-provider-defined-signature>

{
  "amount": 4250,
  "currency": "GBP",
  "reference": "ORDER-8472",
  "return_uri": "https://merchant.example/payments/return"
}
```

The header names and signature input above are placeholders. Do not infer a production signing procedure from this example.

### Signing controls

- Sign on the backend, never in client-side code.
- Preserve the exact body bytes used to calculate the signature.
- Select keys by an explicit key identifier where the provider supports rotation.
- Store private keys in an appropriate secrets or key-management service.
- Log the key identifier, request ID and outcome—but never log private keys, access tokens or sensitive payment data.

After a successful response, store the provider payment ID and the returned authorisation action in the same logical operation where practical.

## 4. Send the user for authorisation

Return only the information the client needs to continue the authorisation journey. Depending on the provider, the next action might be a browser redirect, an application-to-application hand-off, a QR journey or an embedded component.

```json
{
  "order_id": "ORDER-8472",
  "payment_state": "PENDING_AUTHORISATION",
  "next_action": {
    "type": "redirect",
    "uri": "https://authorisation.example/session/abc123"
  }
}
```

Validate a returned URI according to the provider contract. Do not allow an untrusted client to substitute an arbitrary authorisation or return URI.

### Treat the return redirect as navigation

When the bank returns the user to your application, use the redirect to resume the interface. Do not mark the order as paid solely because the browser returned successfully.

Instead:

1. identify the order using protected server-side context;
2. read the latest internal payment state;
3. request the provider state if reconciliation is needed; and
4. show a pending screen until a configured success condition is met.

## 5. Receive and verify webhooks

Assume webhook deliveries can be duplicated, delayed, reordered or retried. Authenticate an event before using any value from its body to mutate payment state.

Recommended processing order:

1. Read the raw request body.
2. Extract the provider's signature and key identifier.
3. Resolve the key only from a trusted key set or configured trust source.
4. Verify the signature using the provider's documented inputs and algorithm.
5. Enforce any documented freshness or replay window.
6. Parse and validate the event schema.
7. Check whether the event ID was already processed.
8. Lock or conditionally update the payment record.
9. Apply only an allowed state transition.
10. Store the event ID and payment update atomically.
11. Return the documented success response.

```text
if signature_is_invalid(raw_request):
    return 401

event = parse_and_validate(raw_request.body)

begin transaction
    if processed_events.contains(event.id):
        commit
        return 204

    payment = payments.lock(event.payment_id)

    if !transition_allowed(payment.state, event.status):
        record_for_investigation(event)
        commit
        return 204

    payments.update(payment, map_status(event.status))
    processed_events.insert(event.id)
commit

return 204
```

The pseudocode is illustrative. Match response codes and retry behaviour to the provider's webhook contract.

### Keep the acknowledgement path short

Complete only the durable acceptance work before responding. Publish slower work—such as emails, analytics or fulfilment—to an internal queue or transactional outbox. This reduces timeouts that can cause avoidable redelivery.

## 6. Reconcile payment state

Webhooks should be the normal asynchronous path, but they should not be your only recovery mechanism. Use the provider's status endpoint to resolve payments that remain non-terminal beyond an expected interval.

Example reconciliation policy:

| Condition | Action |
| --- | --- |
| User returns while state is pending | Fetch current provider state once, then show the internal result |
| Webhook delivery is delayed | Keep the customer-facing state pending and reconcile in the background |
| Payment remains pending beyond the operational threshold | Schedule a status lookup with bounded backoff |
| Provider and internal terminal states conflict | Prevent automatic regression and raise an operational alert |
| Provider status cannot be retrieved | Preserve the last trusted state and retry according to policy |

Avoid aggressive polling. Define a bounded schedule that respects the provider's rate limits and stops when the payment reaches a terminal state.

## Retry decisions

| Situation | Retry? | Required safeguard |
| --- | --- | --- |
| Connection failed before any response | Yes | Reuse the original idempotency key |
| Request timed out after transmission | Yes | Reuse the original idempotency key; do not assume failure |
| Authentication or signature rejected | No automatic retry | Correct credentials, clock or signing input first |
| Validation failed | No automatic retry | Correct the request without reusing an incompatible payload |
| Rate limited | Yes, if permitted | Honour provider guidance and apply jittered backoff |
| Provider returned a transient server error | Usually | Reuse the original idempotency key and apply bounded backoff |
| Webhook event was already processed | No additional work | Return the provider's documented success response |

Never infer retry safety from an HTTP status alone. Use the provider's documented idempotency semantics and error model.

## Observability

Attach a correlation identifier to the customer order, internal payment, outbound request and webhook processing record.

Capture:

- order ID and provider payment ID;
- idempotency key identifier or a safe hash of it;
- request and correlation IDs;
- previous and new internal states;
- provider event ID and event type;
- signature-verification outcome;
- retry count and terminal outcome; and
- timestamps for creation, authorisation, execution and settlement signals.

Do not record credentials, signing material or unnecessary personal and payment data.

Alert on sustained signature failures, growing webhook lag, unusual duplicate rates, payments stuck in non-terminal states and reconciliation mismatches.

## Sandbox test checklist

Test the complete lifecycle, not only the happy path.

- [ ] A valid payment reaches the configured success state.
- [ ] Repeating the creation request with the same idempotency key does not create a second payment.
- [ ] Reusing a key with a different payload is rejected safely.
- [ ] A rejected or cancelled authorisation reaches the correct terminal state.
- [ ] Returning to the application does not mark a pending payment as successful.
- [ ] A webhook with an invalid signature cannot change state.
- [ ] The same valid webhook delivered twice produces one state change.
- [ ] A late event cannot regress a terminal state.
- [ ] A temporary webhook failure causes safe redelivery.
- [ ] Reconciliation recovers a missed lifecycle notification.
- [ ] Logs contain correlation data without secrets or sensitive payloads.

## Definition of done

The integration is ready for production review when:

- the provider-to-internal status map is documented and approved;
- the idempotency and retry policy is covered by automated tests;
- signing keys are stored and rotated through an approved mechanism;
- webhook authentication fails closed;
- duplicate and reordered events are handled deterministically;
- reconciliation has explicit timing, retry and escalation rules;
- operational dashboards and alerts identify stuck payments; and
- support teams can trace a payment using a non-sensitive correlation identifier.

## Standards references

- [Open Banking UK: Read/Write API v4.0.1](https://openbankinguk.github.io/read-write-api-site3/v4.0.1/)
- [Open Banking UK: Domestic Payments v4.0.1](https://openbankinguk.github.io/read-write-api-site3/v4.0.1/resources-and-data-models/pisp/domestic-payments.html)
- [RFC 7515: JSON Web Signature](https://www.rfc-editor.org/rfc/rfc7515)
- [RFC 9110: HTTP Semantics](https://www.rfc-editor.org/rfc/rfc9110)

## Next guide

Continue with **Webhook Signature Verification and Replay Protection** to turn the acceptance algorithm into a detailed implementation pattern.
