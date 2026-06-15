# Auth

See also → [Authentication](/api/authentication) for token endpoints, APTs, and the `strict` flag · [Authorization](/api/authorization) for ABAC, grants, scopes, and the `AuthorityInterceptor`

**Port:** REST `:3020` · gRPC `:5020`

Handles all authentication flows, token management, personal API keys, and OAuth permission grants. Unlike every other service, `auth` does not follow the standard CRUD pattern for its primary endpoint.

## Collections

| Collection | Path | Purpose |
| --- | --- | --- |
| -- | `/auth` | Token issuance, verification, logout, health check, and ABAC policy evaluation |
| APTs | `/auth/apts` | Long-lived Auth Personal Tokens (API keys) |
| Grants | `/auth/grants` | OAuth permission grants |

## `auth` — Token Endpoint

The primary auth endpoint is not a CRUD collection. It exposes named operations:

| Operation | HTTP | Description |
| --- | --- | --- |
| `token` | `POST /auth/token` | Issue a JWT (public — no auth header required) |
| `verify` | `GET /auth/verify` | Decode the current token (JWT or APT) and return its claims |
| `logout` | `GET /auth/logout` | Invalidate the current session (blacklists it) |
| `check` | `POST /auth/check` | Validate a TOTP (one-time password) code |
| `can` | `POST /auth/can` | ABAC policy evaluation via `abacl` |

> `POST /auth/token` is `@IsPublic()` — no `Authorization` header required.

## `auth/apts` — Auth Personal Tokens

Long-lived API keys scoped to a set of OAuth scopes. Use for service-to-service authentication or CI/CD automation.

### Create Fields

Only `name` is required; the rest are optional. See [Authentication → APT fields reference](/api/authentication#apt-fields-reference) for the full list.

| Field | Required | Type | Description |
| --- | :---: | --- | --- |
| `name` | ✅ | `string` | Human-readable label (e.g. `ci-bot`) |
| `scopes` | | `string[]` | OAuth scopes this APT is authorized to use |
| `subjects` | | `string[]` | ABAC subjects for the token |
| `strict` | | `boolean` | Require a valid `x-api-key` header (default `false`) |
| `expires_at` | | `number` | Expiry as Unix milliseconds |
| `coworkers` | | `string[]` | Additional coworker client IDs |

### Key Behaviors

- APTs are revocable — delete the record to invalidate immediately.
- Always create APTs with the minimum required scopes.
- APTs appear in `Authorization: Bearer <apt>` headers, the same as JWTs.

## `auth/grants` — Permission Grants

OAuth permission grants define what actions a subject (user or role) may perform on a resource within a domain.

### Required Grant Fields

| Field | Required | Type | Description |
| --- | :---: | --- | --- |
| `action` | ✅ | `Action` enum | `read`, `write`, `manage`, or a custom action |
| `subject` | ✅ | string | Subject identifier — `{username}@{domain}` (e.g. `guest@example.com`) |
| `object` | ✅ | `Resource` | Resource identifier (e.g. `identity:users` or `identity:*`) |

### Grant Behaviors

- Grants are evaluated by `PolicyGuard` on every authenticated request.
- Subject format: `{username}@{domain}` — the domain suffix is required in grants but **not** stored in `identity/users.subjects`.
- Use `POST /auth/can` to test whether a subject has a specific grant before making changes.
