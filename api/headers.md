# Request Headers

This is the canonical reference for every request header the Wenex Platform gateway
recognizes. Headers are read into the per-request `Metadata` object by
`MetadataInterceptor` (and a few feature interceptors) and propagated to services
over gRPC. Anything not listed here is ignored.

> Tenant, client, and user context come entirely from the **JWT**, not from headers.
> There is no `x-domain` or `x-client-id` header — see
> [Authentication](/api/authentication) and [Coworkers Space](/getting-started/overview/key-concepts/coworkers-space).

## Authentication

| Header | Value | Required | Description |
|---|---|---|---|
| `Authorization` | `Bearer <jwt\|apt>` | Yes* | Bearer JWT (`eyJ…`) or APT (`APT-…`). May also be supplied as the `?token=` query param or an `authorization` cookie. *Omitted only on `@IsPublic()` routes. |
| `x-api-key` | base64 AES-encrypted `ApiToken` | Conditional | Required when the token was issued with `strict: true`. Validated by `AuthShield`; see [Authentication → the strict flag](/api/authentication#the-strict-flag-and-x-api-key). |

## Query shaping

| Header | Value | Required | Description |
|---|---|---|---|
| `x-zone` | comma list of `own`, `share`, `group`, `client` | No | ABAC ownership zones applied to reads. Default `own,share`. Also settable via `?zone=`. See [Access Control](/getting-started/overview/key-concepts/access-control). |
| `x-exclude-soft-delete-query` | truthy | No | Include soft-deleted documents (those with a non-null `deleted_at`) in read results. |
| `x-naming-convention` | `snake_case` (default), `camelCase`, … | No | Naming convention applied to request and response body fields (via `naming-conventions-modeler`). |
| `x-no-api-response` | truthy | No | Skip the standard response envelope and return the handler payload as-is. |
| `x-ignore-change-projection` | truthy | No | Suppress the `x-change-projection-value` response header the projection interceptor otherwise sets. |

## Tracing & locale

| Header | Value | Required | Description |
|---|---|---|---|
| `x-request-id` | uuid | No | Correlation / trace id. Auto-generated if absent. |
| `x-lang` | language code (`en`, `fa`, …) | No | Locale for response localization. |
| `x-user-agent` | string | No | Client user agent; recorded in audit metadata. |
| `x-user-ip` | ip | No | Real client IP. Honored only when a valid `x-api-key` is present (a trusted backend forwarding the end-user IP); otherwise the socket IP is used. |
| `x-at` | ISO datetime | No | Operation "as-of" timestamp. Defaults to the server's current time. |

## ABAC introspection — `POST /auth/can`

| Header | Value | Required | Description |
|---|---|---|---|
| `x-can-with-policies` | truthy | No | Include the matching policy list in the `can` response. |
| `x-can-with-id-policies` | truthy | No | Include id-level policies in the `can` response. |

## Distributed transactions — `essential` sagas

| Header | Value | Required | Description |
|---|---|---|---|
| `x-saga-session` | saga id | No | Enlist the operation in an existing saga / transaction session. |
| `x-saga-ttl` | seconds | No | TTL for a newly created saga session. |

## Domain-specific

| Header | Value | Required | Description |
|---|---|---|---|
| `x-payment-amount` | number | No | Payment amount consumed by `financial` invoice / payment actions. |
| `x-file-data` | JSON `FileData` | No | Upload metadata for `special/files` uploads. |
| `x-stat-type` | `StatType` enum | No | Stat type to record against `special/stats`. |
| `x-stat-learning-rate` | number | No | Learning rate for the stat model. |
| `x-stat-delay-datetime` | datetime | No | Delay / anchor datetime for the stat. |
| `x-stat-additional-props` | JSON `{ obj?, flag? }` | No | Extra stat props — `obj` must be a MongoId, `flag` must match the key pattern. |

## Notes

- **Derived, not accepted from clients.** The platform computes `x-by`, `x-in`,
  `x-token`, `x-scope`, `x-policy`, `x-api-token`, and `x-nested-keys` from the
  validated token and route metadata. Sending them has no effect.
- **Response headers.** The gateway echoes some context back on the response —
  e.g. `x-zone`, `x-at`, `x-change-projection-value`, and `x-no-api-response`.
