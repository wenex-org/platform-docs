# General

**Port:** REST `:3070` · gRPC `:5070`

Shared cross-cutting entities used across multiple domains: audit activities, typed key-value artifacts, threaded comments, calendar events, and BPMN-style workflow state.

## Collections

| Collection | Path | Purpose |
| --- | --- | --- |
| Activities | `/general/activities` | Audit and activity log entries |
| Artifacts | `/general/artifacts` | Typed key-value or JSON-like stored artifacts |
| Comments | `/general/comments` | Threaded comments for posts and tickets |
| Events | `/general/events` | Calendar-like events, meetings, and deadlines |
| Workflows | `/general/workflows` | BPMN-style process state and token history |

## `general/activities`

Append-heavy audit and activity records. Treat as write-once — prefer creating new activities over mutating old ones.

### Required Create Fields

| Field | Required | Type | Description |
| --- | :---: | --- | --- |
| `type` | ✅ | `ActivityType` | `USER`, `SYSTEM`, `EXTERNAL` |
| `message` | ✅ | string | Human-readable summary |

### Optional Fields

| Field | Type | Description |
| --- | --- | --- |
| `source` | string | Originating service or module |
| `details` | object | Structured payload |
| `metadata` | object | Extra structured context |

## `general/artifacts`

Typed key-value storage for generated reports, serialized outputs, exports, and flexible JSON payloads.

### Required Create Fields

| Field | Required | Type | Description |
| --- | :---: | --- | --- |
| `key` | ✅ | string | Artifact key — uppercase, underscore, digit, dash, and dot pattern |
| `type` | ✅ | `ValueType` | `NULL`, `ARRAY`, `OBJECT`, `STRING`, `NUMBER`, `BOOLEAN` |

### Optional Fields

| Field | Type | Description |
| --- | --- | --- |
| `value` | object | Artifact payload |

## `general/comments`

Threaded discussion attached to `content/posts` or `content/tickets`. Comments live in this service, not in the `content` service.

### Required Create Fields

| Field | Required | Type | Description |
| --- | :---: | --- | --- |
| `content` | ✅ | string | Comment body |

### Optional Fields

| Field | Type | Description |
| --- | --- | --- |
| `post` | MongoId | Target `content/posts` document |
| `ticket` | MongoId | Target `content/tickets` document |
| `parent` | MongoId | Parent comment for threading |
| `level` | integer | Nesting depth |
| `status` | `CommentStatus` | `DRAFT`, `ARCHIVED`, `PUBLISHED` |
| `mentions` | MongoId[] | Raw mentioned IDs — not populatable |
| `attachments` | MongoId[] | Raw file IDs — not populatable |
| `reactions` | object[] | Reaction objects |

### Population

| Path | Collection |
| --- | --- |
| `post` | `content/posts` |
| `ticket` | `content/tickets` |
| `parent` | `general/comments` |

## `general/events`

Scheduled meetings, deadlines, reminders, and other bounded events.

### Required Create Fields

| Field | Required | Type | Description |
| --- | :---: | --- | --- |
| `title` | ✅ | string | Event title |
| `s_date` | ✅ | date | Start datetime |
| `e_date` | ✅ | date | End datetime — must be after `s_date` |

### Optional Fields

| Field | Type | Description |
| --- | --- | --- |
| `subtitle` | string | Subtitle |
| `place` | string | Venue label |
| `location` | MongoId | Raw `logistic/locations` ID — not populatable |
| `attendees` | MongoId[] | `identity/profiles` IDs |
| `organizers` | MongoId[] | `career/employees` IDs |
| `category` | string | Category label |
| `correlation` | UUID | Groups recurring event instances |

### Population

| Path | Collection |
| --- | --- |
| `attendees` | `identity/profiles` |
| `organizers` | `career/employees` |

## `general/workflows`

BPMN-style workflow execution state, including overall status and per-token history.

### Required Create Fields

| Field | Required | Type | Description |
| --- | :---: | --- | --- |
| `name` | ✅ | string | Workflow name |
| `status` | ✅ | `WorkflowStatus` | `ready`, `paused`, `failed`, `running`, `completed`, `terminated` |
| `tokens` | ✅ | `WorkflowToken[]` | Execution threads |

### `WorkflowToken` Structure

| Field | Required | Type | Description |
| --- | :---: | --- | --- |
| `id` | ✅ | string | Token identifier |
| `history` | ✅ | `WorkflowState[]` | Ordered state history |
| `parent` | — | string | Parent token ID |
| `locked` | — | boolean | Rollback guard |

Each `WorkflowState` entry has `ref` (BPMN element reference), `status`, optional `name`, and optional `value`.

> Preserve token history carefully — do not guess workflow state transitions or invent BPMN element references.
