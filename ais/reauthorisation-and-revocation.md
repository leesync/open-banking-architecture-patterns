# Consent Expiry, Reauthorisation and Revocation Reference

**Diátaxis type:** Reference

| Condition | Local state | Collection action | User experience |
| --- | --- | --- | --- |
| Consent active and credential valid | `ACTIVE` | Continue within scope | Show last refresh time |
| Credential expired; refresh permitted | `ACTIVE` | Refresh once under policy | Usually no interruption |
| Credential expired; user action required | `REAUTHORISATION_REQUIRED` | Stop scheduled access | Explain why reconnection is needed |
| Consent nearing expiry | `EXPIRING` | Continue until the authorised boundary | Prompt before expiry without implying failure |
| User or provider revoked consent | `REVOKED` | Stop immediately | Confirm disconnection |
| Consent expired | `EXPIRED` | Stop | Offer a new consent journey |
| Scope changed or unsupported | `REVIEW_REQUIRED` | Fail closed for affected resources | Ask the user to review permissions |

## Rules

- Reauthorisation creates or renews authority only through the documented user journey.
- Token refresh must not expand scope.
- Revocation and expiry must win over delayed background jobs.
- Store timestamps, source and correlation IDs for state changes.
- Avoid hard-coding a universal consent duration; regulations, schemes and provider contracts differ.
