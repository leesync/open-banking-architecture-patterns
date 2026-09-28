# How to Reconcile Delayed or Missing Payment Updates

**Diátaxis type:** How-to guide  
**Outcome:** Resolve payments stuck in a non-terminal state without unsafe polling or duplicate side effects

## Procedure

1. Select payments whose state is non-terminal beyond a documented threshold.
2. Claim each record with a lease or conditional update so workers do not reconcile it concurrently.
3. Retrieve the provider's current state using the stored provider payment ID.
4. Map the result through the versioned status adapter.
5. Apply only an allowed transition.
6. Record the previous state, provider state, request ID and evidence time.
7. Dispatch side effects through the same outbox used by webhook processing.
8. Stop scheduling when the payment is terminal or the escalation limit is reached.

## Suggested schedule

Use bounded, jittered intervals that respect provider rate limits. Increase the delay as the payment ages. Do not prescribe one universal timetable: payment rails and provider contracts differ.

## Conflicts

| Observation | Action |
| --- | --- |
| Provider is ahead of internal state | Apply the allowed forward transition |
| Internal terminal state conflicts with provider | Do not regress automatically; escalate |
| Provider resource is temporarily unavailable | Preserve state and retry within policy |
| Provider cannot find the payment after an uncertain create | Keep the outcome unknown until the documented recovery window closes |
| Repeated drift across many payments | Alert as a systemic integration incident |

Reconciliation complements webhooks; it is not a reason to ignore webhook reliability.
