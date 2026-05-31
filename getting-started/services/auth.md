# Auth

See also → [Authentication](/api/authentication) for token endpoints, APTs, and the `strict` flag · [Authorization](/api/authorization) for ABAC, grants, scopes, and the `AuthorityInterceptor`

**Port:** REST `:3020` · gRPC `:5020`

Handles all authentication flows, token management, personal API keys, and OAuth permission grants. Unlike every other service, `auth` does not follow the standard 14-operation CRUD pattern for its primary endpoint.

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
| `token` | `POST /auth/token` | Issue JWT (public — no auth header required) |
| `register` | `POST /auth/register` | Register a new user account |
| `otp` | `POST /auth/otp` | Request a one-time password |
| `verify` | `POST /auth/verify` | Confirm OTP and activate account |
| `repass` | `POST /auth/repass` | Forgot / reset password |
| `oauth` | `POST /auth/oauth` | OAuth (Google / social) login |
| `logout` | `POST /auth/logout` | Invalidate the current token |
| `check` | `GET /auth/check` | Validate whether the current token is still live |
| `can` | `POST /auth/can` | ABAC policy evaluation via `abacl` |

> `POST /auth/token` is `@IsPublic()` — no `Authorization` header required.

## `auth/apts` — Auth Personal Tokens

Long-lived API keys scoped to a set of OAuth scopes. Use for service-to-service authentication or CI/CD automation.

### Required Create Fields

| Field | Required | Type | Description |
| --- | :---: | --- | --- |
| `scopes` | ✅ | `string[]` | OAuth scopes this APT is authorized to use |

### Key Behaviors

- APTs are revocable — delete the record to invalidate immediately.
- Always create APTs with the minimum required scopes.
- APTs appear in `Authorization: Bearer <apt>` headers, the same as JWTs.

## `auth/grants` — Permission Grants

OAuth permission grants define what actions a subject (user or role) may perform on a resource within a domain.

### Required Grant Fields

| Field | Required | Type | Description |
| --- | :---: | --- | --- |
| `action` | ✅ | `Action` enum | `read`, `write`, `manage` |
| `subject` | ✅ | string | Subject identifier (e.g. `guest@example.com`) |
| `resource` | ✅ | string | Resource identifier (e.g. `identity:users`) |
| `domain` | ✅ | string | FQDN of the tenant domain |

### Grant Behaviors

- Grants are evaluated by `PolicyGuard` on every authenticated request.
- Subject format: `{username}@{domain}` — the domain suffix is required in grants but **not** stored in `identity/users.subjects`.
- Use `POST /auth/can` to test whether a subject has a specific grant before making changes.
