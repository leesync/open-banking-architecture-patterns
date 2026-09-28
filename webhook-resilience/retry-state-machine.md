# Webhook Retry State Machine Reference

**Diátaxis type:** Reference

This is an internal consumer state machine; provider delivery policies remain authoritative.

| State | Meaning | Next action |
| --- | --- | --- |
| `RECEIVED` | Request arrived but is not trusted | Verify signature and freshness |
| `REJECTED` | Authentication or schema policy failed | No business processing; alert by threshold |
| `DUPLICATE` | Event ID was already committed | Return documented success |
| `ACCEPTING` | Atomic state update is in progress | Commit or roll back |
| `ACCEPTED` | Event and required state change are durable | Acknowledge; dispatch outbox work |
| `RETRYABLE_FAILURE` | Durable acceptance failed temporarily | Return provider-defined retryable response |
| `QUARANTINED` | Authentic event cannot be applied safely | Preserve evidence and investigate |

## Response decision

Return success only when the provider contract permits it and the event is durably accepted, already accepted, or safely quarantined according to policy. Return a retryable failure when the transaction did not commit. Never return success merely because an in-memory queue accepted the event.
