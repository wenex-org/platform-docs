# MCP Tools Reference

The gateway registers a **fixed set of 12 MCP tools** — 2 core tools plus 10 generic
resource-CRUD tools. There are **no per-collection tools** (e.g. there is no
`find_identity_users`); instead, every resource tool takes a `resource` argument that
selects the target service/collection.

## Core Tools

### `auth_verify`

Verifies the agent's current token (JWT or APT) and returns the decoded claims.

**Call this tool first** before any resource operation to confirm identity and understand
which scopes are available.

**Input:** optional `headers` (object) to override request headers.

**Example output** (the same `JwtToken` shape returned by `GET /auth/verify`):

```json
{
  "type": "access",
  "cid": "64a1b2c3d4e5f6a7b8c9d0e1",
  "uid": "64a1b2c3d4e5f6a7b8c9d0e2",
  "scope": "read:identity:users read:financial:transactions",
  "subject": "agent@example.com",
  "domain": "example.com"
}
```

### `read_documentations`

Loads MCP specification documentation by URI so agents can self-serve context about the
platform's capabilities, resources, and operations.

**Input schema:**

```json
{
  "type": "object",
  "required": ["uri"],
  "properties": {
    "uri": {
      "type": "string",
      "description": "Documentation URI, e.g. docs://core/specification"
    }
  }
}
```

**Available URIs:**

| URI | Content |
|---|---|
| `docs://readme` | Routing entrypoint and load-order rules |
| `docs://core/specification` | Canonical MCP rules, metadata headers, operations |
| `docs://core/resource-specification` | Service and collection catalog |
| `docs://core/auth-specification` | APTs, scopes, grants, subjects, zones |
| `docs://core/agent-guidance` | MongoDB query patterns and Mermaid diagram guidance |
| `docs://core/cross-service-pattern` | Multi-service workflow patterns |
| `docs://service/identity-specification` | Identity service (users, profiles, sessions) |
| `docs://service/financial-specification` | Financial service (accounts, wallets, …) |
| `docs://service/{service}-specification` | One per service — see the list below |

## Resource CRUD Tools

The remaining 10 tools each accept a `resource` argument (the target `service/collection`)
plus the relevant `filter` / `params` / `data` / `headers`. They mirror the REST CRUD
surface:

| Tool | REST equivalent | Purpose |
|---|---|---|
| `count` | `GET /{resource}/count` | Count matching documents |
| `create` | `POST /{resource}` | Create one document |
| `create_bulk` | `POST /{resource}/bulk` | Create many documents |
| `find` | `GET /{resource}` | Find with filter / pagination |
| `find_one` | `GET /{resource}/:id` | Find one by `params.id` (ObjectId) or `params.ref` |
| `update_one` | `PATCH /{resource}/:id` | Update one by id |
| `update_bulk` | `PATCH /{resource}/bulk` | Update many matching a query |
| `delete_one` | `DELETE /{resource}/:id` | Soft-delete one |
| `restore_one` | `PUT /{resource}/:id/restore` | Restore a soft-deleted document |
| `destroy_one` | `DELETE /{resource}/:id/destroy` | Hard-delete one (permanent) |

### The `resource` enum

`resource` is a string enum of `service/collection` values. Most CRUD collections across
the 14 services are addressable; a few entries (e.g. `auth/apts`, `touch/push-histories`)
are intentionally **not** exposed as MCP resources and return a "not implemented" error.

```jsonc
// Example: find active identity users
{
  "resource": "identity/users",
  "filter": { "query": { "status": "active" }, "pagination": { "limit": 10 } }
}
```

Filter shape matches the REST [filter system](../api/filtering.md): `query` (object),
`pagination` (`{ skip ≥ 0, limit 1..50, sort }`), `populate` (`[{ path, model, match,
select, options }]`), `projection`. `find_one` additionally accepts `params: { id?, ref? }`.

## MCP Documentation Spec Files

The `mcp/` directory contains the specification files served by `read_documentations`.
Each spec is a **single file** (there are no compact/extended variants):

```
mcp/
├── readme.md                              # MCP routing guide (not a tool)
├── core/
│   ├── -specification.md                  # Canonical MCP rules
│   ├── resource-specification.md          # Service/collection catalog
│   ├── auth-specification.md              # Auth mechanics
│   ├── agent-guidance.md                  # MongoDB query patterns for agents
│   └── cross-service-pattern.md           # Multi-service workflow patterns
└── service/
    ├── identity-specification.md
    ├── financial-specification.md
    ├── career-specification.md
    └── … (one per service: conjoint, content, context, domain, essential,
           general, logistic, special, thing, touch)
```

> `docs://core/specification` is the canonical URI; on disk it maps to
> `core/-specification.md`. Service URIs are `docs://service/{service}-specification`.
