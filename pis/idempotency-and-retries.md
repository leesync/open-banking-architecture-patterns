# How to Make Payment Creation Retry-Safe

**Diátaxis type:** How-to guide  
**Outcome:** Retry an uncertain request without creating another logical payment

## Choose the operation identity

Generate the idempotency key on the backend from the business operation, not the network attempt.

```text
payment:{merchant_order_id}:{operation_version}
```

Persist the key and a fingerprint of material request fields before calling the provider. The fingerprint can include amount in minor units, currency, beneficiary and payment type.

## Decision procedure

| Situation | Action |
| --- | --- |
| Connection failed before a response | Retry with the same key |
| Timeout after sending | Treat the result as unknown; retry or look up using the same key |
| Same key, same fingerprint | Return or recover the original result |
| Same key, different fingerprint | Reject locally as a conflict |
| Validation or authentication failure | Correct the defect; do not loop automatically |
| Rate limit or transient provider error | Follow provider guidance with bounded exponential backoff and jitter |
| Terminal business rejection | Do not retry the same operation automatically |

An HTTP timeout is not evidence that the provider did nothing. Never create a new key merely because the client repeated a request.

## Storage model

Store:

- idempotency key;
- request fingerprint;
- internal order ID;
- provider payment ID when known;
- processing state;
- original result or a safe pointer to it; and
- retention expiry aligned with the provider and business retry window.

Use a unique constraint on the key. Concurrent requests must converge on one record.

## Recovery

If the provider accepted the request but the response was lost, recover using its idempotency lookup or payment-status interface where available. Escalate an unresolved unknown outcome rather than starting another payment blindly.
