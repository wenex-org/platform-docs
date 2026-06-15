# Essential

**Port:** REST `:3050` · gRPC `:5050`

Manages distributed saga transactions across multiple services. Sagas coordinate multi-step operations (e.g. a business creation that touches `logistic/locations`, `financial/accounts`, and `context/configs` atomically). Saga records and their per-step stages are stored in MongoDB; if a saga's TTL expires before it is committed, the `watcher` worker triggers compensation for crash recovery.

## Collections

| Collection | Path | Purpose |
| --- | --- | --- |
| Sagas | `/essential/sagas` | Top-level orchestration record |
| Saga Stages | `/essential/saga-stages` | Per-step execution history |

## `essential/sagas`

Tracks the top-level orchestration unit for a long-running or distributed operation.

### Starting a Saga (Recommended)

Use the platform REST `start` endpoint — not the raw CRUD create:

```
POST /essential/sagas/start
{ "ttl": 3600 }
```

The Platform manages `job`, `state`, and `session` automatically. This is the correct path for user-facing saga orchestration.

### Raw Create Fields (Advanced / Admin Use)

| Field | Required | Type | Description |
| --- | :---: | --- | --- |
| `ttl` | ✅ | number | Saga lifetime in seconds |
| `job` | ✅ | UUID | Saga job identifier — must be provided explicitly |
| `state` | ✅ | `SagaState` | `AWAITING`, `ABORTED`, `COMMITTED` |
| `session` | ✅ | hex string | Associated session identity |

### Special Operations (Platform REST only — no MCP equivalent)

| Operation | HTTP | Purpose |
| --- | --- | --- |
| `start` | `POST /essential/sagas/start` | Initiate a saga (preferred) |
| `add` | `POST /essential/sagas/add` | Append a stage to an in-flight saga |
| `abort` | `GET /essential/sagas/:id/abort` | Abort an in-flight saga |
| `commit` | `GET /essential/sagas/:id/commit` | Commit a completed saga |

## `essential/saga-stages`

Records one step within a saga, including what was attempted and what came back.

> `essential/saga-stages` has **no MCP tools**. Stage data can only be read or written via platform REST.

### Required Create Fields

| Field | Required | Type | Description |
| --- | :---: | --- | --- |
| `saga` | ✅ | MongoId | Parent saga ID |
| `bucket` | ✅ | `Collection` enum | Target collection for this stage |
| `action` | ✅ | `SagaStageAction` | `COUNT_DOCUMENTS`, `INSERT_MANY`, `FIND_ONE`, `FIND`, `FIND_ONE_AND_UPDATE`, `FIND_ONE_AND_DELETE`, `UPDATE_MANY` |
| `req` | ✅ | object | Captured request payload |

### Population

`essential/saga-stages` supports `saga` → `EssentialSaga` plus common population.

## Key Behaviors

- Sagas are typically backend-driven. Client code starts them using `start`, then the Platform's `watcher` worker handles timeout compensation automatically.
- If a saga's TTL expires before it is committed, the `watcher` worker triggers compensation (rollback) of all recorded stages.
- The primary cross-service consumer is `financial/transactions` — every transaction is saga-linked.
