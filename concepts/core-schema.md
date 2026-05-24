# Core Schema

Every document stored in the Wenex Platform MongoDB carries a fixed set of base fields in addition to its domain-specific payload. These fields are injected or enforced by the Platform and are never set directly by the client at read time.

---

## Base Fields

| Field | Type | Set by | Description |
| --- | --- | --- | --- |
| `id` | `string` (MongoId) | Platform | Unique document identifier |
| `owner` | `string` (MongoId) | Platform (auto) | The user who created the document |
| `shares` | `string[]` (MongoIds) | Client | Explicit user-level sharing list |
| `groups` | `string[]` (FQDN / email domain) | Client | Group-level access by email domain |
| `clients` | `string[]` (MongoIds) | Platform (auto) + Client | OAuth clients that can access the document |
| `created_at` | `Date` | Platform (auto) | Timestamp of initial creation |
| `updated_at` | `Date` | Platform (auto) | Timestamp of last update |
| `deleted_at` | `Date \| null` | Platform (auto) | Set on soft-delete; `null` when the document is active |
| `version` | `number` | Platform (auto) | Optimistic concurrency counter, incremented on each update |

---

## Ownership Injection

When a document is created, the Platform automatically injects:

- The authenticated user's `sub` claim → `owner`
- The token's `client_id` → `clients[]`

A client may supply additional `client_id` values in `clients[]` at creation time to grant immediate access to coworkers or partner applications.

---

## Soft Delete vs. Hard Delete

All collections use soft-delete by default. The `deleted_at` field controls document visibility:

| State | `deleted_at` value | Visible in queries |
| --- | --- | --- |
| Active | `null` | Yes |
| Soft-deleted | ISO timestamp | No (filtered out) |
| Hard-deleted | — | Document removed from MongoDB |

Three lifecycle operations control this:

| Operation | HTTP | Effect |
| --- | --- | --- |
| `deleteOne` / `deleteById` | `DELETE /:id` | Sets `deleted_at` |
| `restoreOne` / `restoreById` | `PUT /:id/restore` | Clears `deleted_at` |
| `destroyOne` / `destroyById` | `DELETE /:id/destroy` | Permanently removes the document |

Hard delete (`destroy`) requires the `Manage` scope, not just `Write`.

---

## Versioning

The `version` field is an integer incremented by the Platform on every write. It enables optimistic concurrency: clients can pass a `version` value on `updateOne` to prevent overwriting a document that has changed since it was read.

---

## Example Document

```json
{
  "id": "68fc7a456e8fa60ae29c3d02",
  "owner": "680621e8b3f2a10ae19c4d01",
  "shares": [],
  "groups": ["acme.com"],
  "clients": ["68fc7a456e8fa60ae29c3d02", "71ab2b789f1ea71bf30d4e13"],
  "created_at": "2026-05-01T08:00:00.000Z",
  "updated_at": "2026-05-20T14:30:00.000Z",
  "deleted_at": null,
  "version": 3,
  "...domain fields": "..."
}
```

The `shares`, `groups`, and `clients` fields drive the ABAC zone filter on every read. See [ABAC](./abac) for how the Platform uses them.
