# Tutorial: Build and Test a Local Webhook Receiver

**Diátaxis type:** Tutorial  
**Outcome:** Observe a signed event being accepted once and a duplicate producing no second side effect

This learning exercise uses a shared-secret HMAC because it is easy to run locally. It does **not** define the security scheme for any provider. Production code must implement the provider's documented signature format, key distribution and acknowledgement contract.

## 1. Create the receiver

Create a small HTTP service that:

1. reads the request body as bytes;
2. calculates `HMAC-SHA256(secret, raw_body)`;
3. compares the supplied and expected values with a constant-time comparison;
4. rejects an invalid signature before parsing business fields; and
5. stores each accepted `event_id` once.

Pseudocode:

```text
receive(raw_body, signature):
  expected = hmac_sha256(TEST_SECRET, raw_body)

  if !constant_time_equal(signature, expected):
      return 401

  event = parse_json(raw_body)

  if event_id_exists(event.event_id):
      return 204

  insert_event_id_and_apply_effect_atomically(event)
  return 204
```

Use an in-memory set only for this exercise. Production deduplication needs durable storage and a uniqueness constraint.

## 2. Send a valid event

Use this payload without changing its bytes:

```json
{"event_id":"evt-001","payment_id":"pay-123","status":"processing"}
```

Calculate the HMAC with a test secret and send it in a placeholder `Provider-Signature` header. Confirm that the receiver:

- returns the chosen success response;
- stores `evt-001`; and
- changes the test payment once.

## 3. Deliver the same event again

Send the identical request a second time. The receiver should acknowledge it but perform no second business action. This demonstrates idempotent consumption: delivery may be at least once, while the business effect is once.

## 4. Tamper with one byte

Change `processing` to `succeeded` without calculating a new signature. The receiver must reject the request and leave state unchanged.

## 5. Reorder two valid events

Send a terminal event, then an older processing event. Add a transition rule that prevents a terminal state from returning to a non-terminal state. Confirm that the stale event is recorded for investigation without regressing the payment.

## What you learned

- Verification depends on the exact signed material.
- Authenticity and freshness are different from deduplication.
- A valid duplicate should not duplicate side effects.
- Event order cannot be assumed.
- The HTTP response is not enough; tests must inspect stored state.

## Move to production guidance

Continue with:

- [Verify and process a signed payment webhook](../webhook-resilience/verify-signed-payment-webhook.md)
- [Replay, duplicate delivery and event ordering](../webhook-resilience/replay-and-deduplication.md)
- [Webhook retry state machine](../webhook-resilience/retry-state-machine.md)
