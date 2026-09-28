# Payment Status Mapping Reference

**Diátaxis type:** Reference  
**Purpose:** Define stable internal meanings independently of any provider vocabulary

| Internal state | Meaning | Terminal? | Allowed next states |
| --- | --- | --- | --- |
| `CREATED` | No provider payment exists yet | No | `PENDING_AUTHORISATION`, `FAILED` |
| `PENDING_AUTHORISATION` | User action is required or incomplete | No | `PROCESSING`, `FAILED`, `CANCELLED` |
| `PROCESSING` | Authorised or accepted, but configured success is not confirmed | No | `SUCCEEDED`, `FAILED`, `CANCELLED` |
| `SUCCEEDED` | The business-defined success signal has been received | Yes | none without an explicit corrective workflow |
| `FAILED` | A terminal failure or rejection occurred | Yes | none |
| `CANCELLED` | The payment was cancelled through an authorised path | Yes | none |

## Adapter requirements

For every provider and scheme, record:

- provider status and version;
- internal state;
- whether the signal describes authorisation, execution or settlement;
- permitted previous states;
- terminality;
- customer message;
- retry or reconciliation action; and
- evidence source in the provider documentation.

Do not equate a successful redirect with payment success. Do not assume that “accepted,” “executed” and “settled” mean the same thing. Unknown statuses must be quarantined or mapped to a safe non-terminal condition until reviewed.

## Illustrative UK Open Banking values

Values such as `Pending`, `AcceptedSettlementInProcess`, `AcceptedSettlementCompleted` and `Rejected` may appear in a specific API version. Treat mappings as adapter configuration, not universal semantics. Consult the current [Open Banking UK Read/Write API](https://openbankinguk.github.io/read-write-api-site3/v4.0.1/).
