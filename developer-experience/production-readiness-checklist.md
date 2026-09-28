# Production-Readiness Checklist

**Diátaxis type:** Reference

## Trust and credentials

- [ ] Private keys and client credentials remain server-side.
- [ ] Algorithms and trusted key sources are configured explicitly.
- [ ] Key rotation and emergency rollback have been tested.
- [ ] Secrets, tokens and sensitive payloads are excluded from logs.

## Payment creation

- [ ] One logical payment uses one stable idempotency key.
- [ ] A reused key with a different fingerprint is rejected.
- [ ] Timeouts are treated as unknown outcomes.
- [ ] Provider status is mapped through a versioned adapter.

## Consent and account data

- [ ] Requested scope is minimal and purpose-bound.
- [ ] Consent, credential and local grant states are separate.
- [ ] Expiry and revocation stop scheduled collection.
- [ ] Pagination, freshness and provenance are tested.

## Webhooks

- [ ] Required raw request material is preserved.
- [ ] Authentication occurs before business fields affect state.
- [ ] Freshness and durable deduplication are both enforced.
- [ ] State changes and event recording are atomic.
- [ ] Late events cannot regress terminal state.
- [ ] Slow side effects use a durable queue or outbox.

## Operations

- [ ] Reconciliation detects stuck or missing updates.
- [ ] Alerts cover verification failures, lag, duplicates and drift.
- [ ] Support can trace a transaction using non-sensitive identifiers.
- [ ] Runbooks define retry, replay and escalation authority.
- [ ] Sandbox tests cover failure, concurrency and key rotation.
