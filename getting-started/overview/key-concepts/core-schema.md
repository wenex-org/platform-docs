# Core Schema

Every document stored in the Wenex Platform MongoDB carries a fixed set of base fields in addition to its domain-specific payload. These fields are injected or enforced by the Platform and are never set directly by the client at read time.

## Identity & Access Fields

| Field | Type | Required | Set by | Description |
| --- | --- | :---: | --- | --- |
| `id` | `string` (MongoId) | ✅ | Platform | Unique document identifier (alias of `_id`) |
| `ref` | `string` | — | Client | Optional external reference; unique within collection when used |
| `owner` | `string` (MongoId) | ✅ | Platform (auto) | The user, app, or client that created the document |
| `shares` | `string[]` (MongoIds) | — | Client | Explicit user-level sharing list |
| `groups` | `string[]` (FQDN / app ID) | — | Platform (auto) + Client | Group-level access values; auto-populated from token `aid` and `domain` |
| `clients` | `string[]` (MongoIds) | ✅ | Platform (auto) + Client | Client IDs with access; auto-populated from token `cid` and coworkers |

## Lifecycle Timestamps & Actors

Every lifecycle transition records both **when** it happened and **who/where** it was triggered. The `_by` fields are MongoIds referencing `identity/users` and the `_in` fields reference `domain/apps` or `domain/clients`.

| Field | Type | Required | Description |
| --- | --- | :---: | --- |
| `created_at` | `Date` | ✅ | Timestamp of initial creation |
| `created_by` | `string` (MongoId) | ✅ | User/app/client that created the document (`uid ?? aid ?? cid`) |
| `created_in` | `string` (MongoId) | ✅ | App or client context at creation time (`aid ?? cid`) |
| `updated_at` | `Date` | — | Timestamp of last update |
| `updated_by` | `string` (MongoId) | — | User/app/client that last updated the document |
| `updated_in` | `string` (MongoId) | — | App or client context at last update time |
| `deleted_at` | `Date \| null` | — | Set on soft-delete; absent when the document is active |
| `deleted_by` | `string` (MongoId) | — | User/app/client that soft-deleted the document |
| `deleted_in` | `string` (MongoId) | — | App or client context at soft-delete time |
| `restored_at` | `Date` | — | Set on restore; absent when not yet restored |
| `restored_by` | `string` (MongoId) | — | User/app/client that restored the document |
| `restored_in` | `string` (MongoId) | — | App or client context at restore time |

## Metadata & Relations Fields

| Field | Type | Required | Description |
| --- | --- | :---: | --- |
| `identity` | `string` (MongoId) | — | Polymorphic single-relation field; target collection varies by context |
| `relations` | `string[]` (MongoIds) | — | Polymorphic multi-relation field; target collection varies by context |
| `description` | `string` | — | Short searchable text summary; indexed for text search |
| `version` | `string` (SemVer) | ✅ | Document version string (default `"1.0.0"`), incremented by the client on meaningful changes |
| `props` | `object` | — | Free-form JSON schema extension; size-limited |
| `tags` | `string[]` | — | Searchable tag list |
| `rand` | `string` (number-string) | ✅ | Platform-managed random helper value |
| `timestamp` | `string` (number-string) | ✅ | Platform-managed numeric timestamp string |

> **Note on polymorphic fields:** `identity` and `relations` hold MongoIds that may point to any Platform collection. The target collection is determined by application context, not by the schema. Do not assume a fixed target.

## Ownership Injection

When a document is **created**, the `OwnershipInterceptor` automatically derives and sets ownership fields from the JWT token — the client does not supply them:

| Field | Derived from token |
| --- | --- |
| `owner` | `uid ?? aid ?? cid` — user ID if present, otherwise app ID, otherwise client ID |
| `groups` | Deduplicated `[aid, domain]` — app ID (if present) plus the token's domain |
| `clients` | Deduplicated `[cid, ...coworkers]` — token client ID plus all coworker client IDs |
| `created_by` | `uid ?? aid ?? cid` |
| `created_in` | `aid ?? cid` |

When a document is **updated**, the interceptor sets `updated_by` and `updated_in` using the same token-derived logic. It does not touch `owner`, `groups`, or `clients` unless the client explicitly includes them in the request body and the ABAC grant allows it.

A client may pass additional `client_id` values in `clients[]` at creation time to grant immediate read access to other applications; the interceptor merges them with the auto-injected coworker IDs.

## Soft Delete vs. Hard Delete

All collections use soft-delete by default. The `deleted_at` field controls document visibility:

| State | `deleted_at` value | Visible in queries |
| --- | --- | --- |
| Active | absent / `null` | Yes |
| Soft-deleted | ISO timestamp | No (filtered out by default) |
| Hard-deleted | — | Document removed from MongoDB |

Use `x-exclude-soft-delete-query: true` on read requests to include soft-deleted records.

Three lifecycle operations (six methods) control document state:

| Operation | HTTP | Effect |
| --- | --- | --- |
| `deleteOne` / `deleteById` | `DELETE /:id` | Sets `deleted_at`, `deleted_by`, `deleted_in` |
| `restoreOne` / `restoreById` | `PUT /:id/restore` | Clears `deleted_at`; sets `restored_at`, `restored_by`, `restored_in` |
| `destroyOne` / `destroyById` | `DELETE /:id/destroy` | Permanently removes the document |

Hard delete (`destroy`) requires the `Manage` scope. Restore requires `Write` scope.

The `owner`, `shares`, `groups`, and `clients` fields drive the ABAC zone filter on every read. See [ABAC](./access-control) for how the Platform uses them.
