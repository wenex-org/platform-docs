# Domain

**Port:** REST `:3030` · gRPC `:5030`

Manages tenant domains, OAuth application definitions, and client registrations. Every token and document is scoped to a client context defined in this service.

## Collections

| Collection | Path | Purpose |
| --- | --- | --- |
| Apps | `/domain/apps` | OAuth application definitions under a client |
| Clients | `/domain/clients` | Top-level tenant / OAuth client records |

## `domain/apps`

An application registered under a client. Apps represent runtime surfaces such as web, mobile, desktop, or application contexts.

### Required Create Fields

| Field | Required | Type | Description |
| --- | :---: | --- | --- |
| `type` | ✅ | `AppType` | `WEB`, `MOBILE`, `DESKTOP`, `APPLICATION` |
| `cid` | ✅ | MongoId | Parent `domain/clients` ID |
| `status` | ✅ | `Status` | `ACTIVE`, `INACTIVE` |

### Optional Fields

| Field | Type | Description |
| --- | --- | --- |
| `name` | string | Display name |
| `url` | string | App URL |
| `scopes` | string[] | App-allowed scopes — must be a subset of the parent client's scopes |
| `grant_types` | `GrantType[]` | `password`, `refresh_token`, `client_credential`, `authorization_code` |
| `access_token_ttl` | number | Access token TTL in seconds |
| `refresh_token_ttl` | number | Refresh token TTL in seconds |
| `change_logs` | object[] | Nested version / changelog entries |

### Key Behaviors

- An app cannot expose scopes the parent client does not have.
- Changing `scopes` or TTLs affects downstream token issuance for that app.

## `domain/clients`

The top-level tenant unit. Every token, ABAC scope, and Coworkers relationship originates here.

### Required Create Fields

| Field | Required | Type | Description |
| --- | :---: | --- | --- |
| `name` | ✅ | string | Client display name |
| `plan` | ✅ | `ClientPlan` | `ALUMINUM`, `GOLD`, `PLATINUM` |
| `status` | ✅ | `Status` | `ACTIVE`, `INACTIVE` |
| `grant_types` | ✅ | `GrantType[]` | Allowed OAuth grant types |

### Optional Fields

| Field | Type | Description |
| --- | --- | --- |
| `scopes` | string[] | Available OAuth scopes for this client |
| `coworkers` | MongoId[] | Other client IDs in the same Coworkers space |
| `whitelist` | string[] | Allowed IP addresses |
| `access_token_ttl` | number | Access token TTL in seconds |
| `refresh_token_ttl` | number | Refresh token TTL in seconds |
| `domains` | object[] | FQDN entries used for subject-domain resolution |
| `services` | object[] | External integration configurations (SMS, push, email providers) |

### Platform-Managed Fields (read-only)

| Field | Description |
| --- | --- |
| `api_key` | Platform-generated API key |
| `client_id` | Platform-generated OAuth client ID |
| `client_secret` | Platform-generated OAuth client secret |
| `expiration_date` | Platform-managed expiry |

### Key Behaviors

- Changes to `domains` affect subject-domain resolution for grants — confirm before updating.
- Changes to `coworkers` affect cross-client data visibility immediately.
- `client_secret` is sensitive — never expose it unnecessarily.
