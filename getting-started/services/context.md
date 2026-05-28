# Context

**Port:** REST `:3040` · gRPC `:5040`

Stores application-wide and per-user configuration data. The two collections serve different scopes: `configs` for entity-scoped platform behavior switches, `settings` for user-scoped preferences.

## Collections

| Collection | Path | Purpose |
| --- | --- | --- |
| Configs | `/context/configs` | Entity-scoped configuration (RBAC, CQRS, feature flags) |
| Settings | `/context/settings` | User-scoped preference storage |

## `context/configs`

Stores configuration entries scoped to an entity ID — a client, app, or user. This is also the registry for all CQRS webhook endpoints.

### Required Create Fields

| Field | Required | Type | Description |
| --- | :---: | --- | --- |
| `key` | ✅ | `ConfigKey` | Controlled enum key |
| `eid` | ✅ | MongoId | Target entity ID (`uid`, `aid`, or `cid`) |

### Optional Fields

| Field | Type | Description |
| --- | --- | --- |
| `value` | JSON | Free-form JSON value |
| `status` | `Status` | `ACTIVE` or `INACTIVE` |

### Known `ConfigKey` Values

| Key | Purpose |
| --- | --- |
| `RBAC` | Role-based access control rules for the target entity |
| `CQRS` | CQRS webhook registration — `value.webhook` is the target URL |
| Collection keys | e.g. `auth/apts`, `identity/users`, `financial/wallets` — per-collection config |

### CQRS Registration Example

```typescript
await platform.context.configs.create({
  key: 'CQRS',
  eid: '<client_id>',
  value: { webhook: 'http://my-client.com/cqrs' },
})
```

> Changing `RBAC` config affects permission resolution platform-wide for the target entity. Always confirm before updating.

## `context/settings`

Stores typed user preference records.

### Required Create Fields

| Field | Required | Type | Description |
| --- | :---: | --- | --- |
| `key` | ✅ | string | Setting name (uppercase + underscore pattern) |
| `type` | ✅ | `ValueType` | Declared value type |

### Optional Fields

| Field | Type | Description |
| --- | --- | --- |
| `value` | JSON | Value matching the declared `type` |
| `status` | `Status` | `ACTIVE` or `INACTIVE` |

### `ValueType` Values

`NULL`, `ARRAY`, `OBJECT`, `STRING`, `NUMBER`, `BOOLEAN`

### Key Behaviors

- Use `find_one` before creating a new setting — prefer updating an existing record over creating duplicates.
- Use `?zone=own,client` for current-user preference lookups.
