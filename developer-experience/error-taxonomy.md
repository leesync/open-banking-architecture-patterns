# Integration Error Taxonomy

**Diátaxis type:** Reference

Classify errors by the action an integrator should take, not only by HTTP status.

| Category | Examples | Retry policy | Documentation response |
| --- | --- | --- | --- |
| Authentication | Invalid token, unknown key | No automatic loop | Identify credential or trust configuration |
| Authorisation | Scope missing, consent inactive | No | Explain required permission or user journey |
| Validation | Missing field, invalid format | No | Point to the exact field and constraint |
| Conflict | Idempotency mismatch, invalid state transition | No blind retry | Explain how to retrieve or reconcile existing state |
| Rate limiting | Quota exhausted | Bounded retry if allowed | Expose retry guidance and limits |
| Transient provider | Temporary dependency or server fault | Bounded backoff | Preserve idempotency and correlation IDs |
| Unknown outcome | Timeout after transmission | Reconcile before creating again | Treat as uncertain, not failed |
| Business rejection | Bank or scheme rejection | Usually no | Present safe reason and next user action |
| Webhook authentication | Invalid signature or timestamp | No | Fail closed and alert |
| Unsupported version | Unknown event or schema version | No | Quarantine and upgrade |

## Error object guidance

A developer-facing error should provide a stable code, concise message, correlation ID, field path where relevant, retryability and a link to remediation guidance. Do not expose secrets, internal stack traces or unnecessary personal data.
