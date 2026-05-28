# Identity

**Port:** REST `:3080` · gRPC `:5080`

Core user management service. Stores user accounts, extended profiles, and login sessions. All other services reference users by their `identity/users` MongoId.

## Collections

| Collection | Path | Purpose |
| --- | --- | --- |
| Users | `/identity/users` | Root user account record |
| Profiles | `/identity/profiles` | Extended personal / profile data |
| Sessions | `/identity/sessions` | Active login session records |

## `identity/users`

Root user account. Core access-control and ownership references across the Platform point to this collection.

### Required Create Fields

| Field | Required | Type | Description |
| --- | :---: | --- | --- |
| `status` | ✅ | `Status` | Account lifecycle state: `ACTIVE`, `INACTIVE` |
| `subjects` | ✅ | `string[]` | At least one subject; values do **not** include domain suffix |
| `email` or `phone` | ✅ | `string` | At least one of these two must be present |

### Optional Fields

| Field | Type | Description |
| --- | --- | --- |
| `email` | string | Lowercased email address |
| `phone` | string | Normalized phone number |
| `username` | string | Unique username |
| `password` | string | Write-only password (never returned) |
| `secret` | string | Write-only OTP/TOTP secret (never returned) |
| `tz` | string | IANA timezone |
| `lang` | string | Locale |
| `region` | string | ISO 3166-1 Alpha-2 |

### Key Behaviors

- `password` and `secret` are write-only — they are never returned in responses.
- `subjects` stores values **without** the domain suffix (e.g. `guest`, not `guest@example.com`).
- Modifying `subjects` changes ABAC grant matching behavior.

## `identity/profiles`

Extended personal record linked through core ownership. A user typically has one profile.

### Required Create Fields

| Field | Required | Type | Description |
| --- | :---: | --- | --- |
| `type` | ✅ | `ProfileType` | `REAL`, `LEGAL`, `GOVERN` |
| `gender` | ✅ | `Gender` | `MALE`, `FEMALE`, `UNKNOWN` |

### Optional Fields

| Field | Type | Description |
| --- | --- | --- |
| `state` | `State` | `PENDING`, `APPROVED`, `REJECTED`, `VERIFIED`, `UNKNOWN` |
| `first_name` | string | Given name |
| `last_name` | string | Family name |
| `nickname` | string | Display alias |
| `birthdate` | date | Birth date |
| `national_code` | string | National identifier |
| `cover` | MongoId | Raw `special/files` ID — not populatable |
| `avatar` | MongoId | Raw `special/files` ID — not populatable |
| `gallery` | MongoId[] | Raw `special/files` IDs — not populatable |
| `verified_at` | date | Verification timestamp |
| `verified_by` | MongoId | Verifier user ID |

### Population

Common population only. `cover`, `avatar`, and `gallery` are raw IDs — they are not populatable.

## `identity/sessions`

Login/session record with origin and expiration metadata.

### Required Create Fields

| Field | Required | Type | Description |
| --- | :---: | --- | --- |
| `ip` | ✅ | string | Valid IPv4 or IPv6 address |
| `agent` | ✅ | string | User-agent string |
| `expiration` | ✅ | number | Unix timestamp |

### Key Behaviors

- Sessions are soft-deleted on logout.
- The `cleaner` worker hard-deletes expired sessions.
- Use `?zone=own,client` to list a user's own sessions.
