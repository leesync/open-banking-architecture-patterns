# Replay, Duplicate Delivery and Event Ordering

**Diátaxis type:** Explanation

These are related but different failure modes.

**Replay** is the reuse of an authentic message outside its intended context or time. A signed timestamp, nonce or provider-defined freshness rule may limit it.

**Duplicate delivery** is often legitimate. A provider may resend an event because the acknowledgement was lost. Durable event-ID deduplication prevents the business effect from running twice.

**Out-of-order delivery** occurs when a newer lifecycle event arrives before an older one. A state-transition policy prevents the older event from regressing trusted state.

No single control solves all three:

| Control | Replay | Duplicate | Reordering |
| --- | --- | --- | --- |
| Signature verification | Proves integrity/authenticity only | No | No |
| Timestamp or nonce | Helps | No | No |
| Unique event ID | Some protection | Yes | No |
| Monotonic transition rules | No | Limits duplicate effects | Yes |
| Reconciliation | Detects drift | Detects gaps | Restores authoritative state |

The safe processing order is: retain signed material, authenticate, check freshness, validate schema, deduplicate durably, apply an allowed transition atomically, then acknowledge.
