# Authentication

Wenex Platform supports two token types for API access.

| | JWT | APT (Auth Personal Token) |
|---|---|---|
| **Lifetime** | Short, configurable | Long, revocable |
| **Use case** | Interactive user sessions | Server-to-server, CI/CD, AI agents |
| **Bearer prefix** | `eyJ…` | `APT-…` |
| **Backed by** | Signed JWT (RS256) | Redis — key `auth:apt:<suffix>` |
| **Strict support** | ✅ | ✅ |

Both are submitted as `Authorization: Bearer <token>`. Routes decorated with `@IsPublic()` are the only exceptions — currently just `POST /auth/token`.

See also → [Authorization](/api/authorization) for ABAC, scopes, grants, and the `AuthorityInterceptor`.

## Quick Start

```bash
# 1. Get your first token
curl -X POST http://localhost:3010/auth/token \
  -H "Content-Type: application/json" \
  -d '{
    "grant_type": "password",
    "client_id": "YOUR_CLIENT_ID",
    "username": "user@example.com",
    "password": "your-password"
  }'

# 2. Use the access_token in every API call
TOKEN="eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9..."
curl http://localhost:3010/identity/users \
  -H "Authorization: Bearer $TOKEN"
```

You need a `client_id` (OAuth application ID) — ask your admin if you don't have one.

### Which token type?

| Scenario | Use |
|---|---|
| User logging in via browser or mobile | JWT — `password` grant |
| Backend service calling Platform | JWT — `client_credential` grant, or APT |
| CI/CD pipeline | APT (personal token) |
| Long-lived server with strict security | APT or JWT with `strict: true` + API key |

## POST /auth/token — Issue a token

The only public endpoint in the platform. Exchanges credentials for a JWT access token (and optionally a refresh token).

### Grant types

| `grant_type` | Required fields | Typical caller |
|---|---|---|
| `password` | `username`, `password`, `client_id` | End-user login |
| `refresh_token` | `refresh_token`, `client_id` | Session rotation |
| `client_credential` | `client_id`, `client_secret` | Machine-to-machine |
| `authorization_code` | `code`, `client_id` | OAuth code exchange |

### Request body

```typescript
{
  grant_type: GrantType;         // required — one of the four above

  // oauth
  client_id: string;             // required
  client_secret?: string;        // required for client_credential
  app_id?: string;

  // password / otp
  username?: string;             // username, email, or phone
  password?: string;             // password or OTP

  // refresh
  refresh_token?: string;

  // authorization_code
  code?: string;
  response_type?: ResponseType;
  state?: string;

  // token shaping
  strict?: boolean;              // if true, issued JWT carries strict: true
  domain?: string;
  scopes?: Scope[];              // restrict the token to a subset of allowed scopes
  subjects?: string[];           // override ABAC subjects
  coworkers?: string[];          // additional coworker client IDs
  source?: AuthSource;
}
```

### Example — password grant

```bash
curl -X POST http://localhost:3010/auth/token \
  -H "Content-Type: application/json" \
  -d '{
    "grant_type": "password",
    "client_id": "64a1b2c3d4e5f6a7b8c9d0e1",
    "username": "admin@example.com",
    "password": "secret",
    "scopes": ["read:identity:users", "write:content:notes"]
  }'
```

### Example — client credentials

```bash
curl -X POST http://localhost:3010/auth/token \
  -H "Content-Type: application/json" \
  -d '{
    "grant_type": "client_credential",
    "client_id": "64a1b2c3d4e5f6a7b8c9d0e1",
    "client_secret": "app-secret"
  }'
```

No `refresh_token` is returned for client credentials — request a new token when it expires.

### Example — refresh token

```bash
curl -X POST http://localhost:3010/auth/token \
  -H "Content-Type: application/json" \
  -d '{
    "grant_type": "refresh_token",
    "client_id": "64a1b2c3d4e5f6a7b8c9d0e1",
    "refresh_token": "eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9..."
  }'
```

### Response

```json
{
  "data": {
    "token_type": "Bearer",
    "access_token": "eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9...",
    "refresh_token": "eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9...",
    "expires_in": 3600,
    "scope": "read:identity:users write:identity:users",
    "domain": "example.com",
    "subject": "admin@example.com"
  }
}
```

| Field | Description |
|---|---|
| `access_token` | JWT to use in `Authorization: Bearer` |
| `refresh_token` | Present when `grant_type` supports session rotation |
| `expires_in` | Seconds until the access token expires |
| `scope` | Space-separated scopes actually granted (may be narrower than requested) |
| `domain` | Tenant domain of this session |
| `subject` | ABAC subject string embedded in the token |
| `coworker` | Space-separated coworker client IDs, if any |
| `session` | Session record ID (present only on certain grant types) |

### Error responses

| Status | Message | Cause |
|---|---|---|
| `401 Unauthorized` | `"username or password is invalid"` | Wrong credentials |
| `400 Bad Request` | `"client_id is required"` | Missing required field |
| `429 Too Many Requests` | `"too many failed login attempts from this IP"` | Rate limit exceeded |

### Token flow diagram

```mermaid
sequenceDiagram
    Client->>Token Endpoint: POST /auth/token<br/>(credentials + grant_type)
    Token Endpoint->>Auth Service: Validate credentials<br/>Create session
    Auth Service->>Token Endpoint: Session record created
    Token Endpoint->>Crypto: Sign JWT with RS256
    Crypto->>Token Endpoint: Signed JWT
    Token Endpoint->>Client: access_token + refresh_token<br/>+ expires_in
```

## GET /auth/verify — Inspect the current token

Decodes a JWT or resolves an APT from Redis and returns its claims. Useful for debugging, session introspection, or confirming scopes before making downstream calls.

```bash
curl http://localhost:3010/auth/verify \
  -H "Authorization: Bearer $TOKEN"
```

### Response — JWT claims (`JwtToken`)

```json
{
  "data": {
    "type": "access",
    "strict": false,
    "cid": "64a1b2c3d4e5f6a7b8c9d0e1",
    "uid": "64a1b2c3d4e5f6a7b8c9d0e2",
    "aid": "64a1b2c3d4e5f6a7b8c9d0e3",
    "scope": "read:identity:users write:identity:users",
    "domain": "example.com",
    "subject": "admin@example.com",
    "tz": "UTC",
    "lang": "en",
    "session": "64a1b2c3d4e5f6a7b8c9d0e4",
    "client_id": "64a1b2c3d4e5f6a7b8c9d0e1",
    "coworker": "64a1b2c3d4e5f6a7b8c9d0e5 64a1b2c3d4e5f6a7b8c9d0e6"
  }
}
```

### Claims reference

| Claim | Type | Description |
|---|---|---|
| `type` | `"access"` \| `"refresh"` | Token class — access tokens call APIs, refresh tokens cannot |
| `strict` | `boolean` | API-key enforcement flag — see [The strict flag](#the-strict-flag-and-x-api-key) |
| `cid` | `string` | Client ObjectId — the OAuth application that owns this session |
| `aid` | `string?` | App ID, present when issued for a registered app |
| `uid` | `string?` | Authenticated user ObjectId |
| `scope` | `string` | Space-separated `Scope` values granted to this token |
| `domain` | `string` | Tenant domain for this session |
| `subject` | `string` | Space-separated ABAC subjects (used by `PolicyGuard`) |
| `tz` | `string` | Timezone string (e.g. `"UTC"`, `"Asia/Tehran"`) |
| `lang` | `string` | Language code (e.g. `"en"`, `"fa"`) |
| `session` | `string` | Session record ID — used for logout and blacklisting |
| `client_id` | `string` | OAuth client ObjectId (same as `cid` for most flows) |
| `coworker` | `string?` | Space-separated coworker client IDs |

::: info APT tokens
When the Bearer value starts with `APT-`, `verify` resolves the APT from Redis (`auth:apt:<suffix>`), decrypts it with AES, and returns the same `JwtToken` shape. The caller sees no difference.

```bash
curl http://localhost:3010/auth/verify \
  -H "Authorization: Bearer APT-xxxxxxxxxxxxxxxxxxxxxxxx"
```
:::

## GET /auth/logout — Invalidate the session

Deletes the session record associated with the current token. The blacklist worker then adds the `session` ID to Redis, causing subsequent requests using that token to be rejected by `AuthGuard`.

Works identically for JWT and APT bearer tokens.

```bash
curl http://localhost:3010/auth/logout \
  -H "Authorization: Bearer $TOKEN"
```

**Response:**

```json
{ "result": "OK" }
```

### Logout sequence

```mermaid
sequenceDiagram
    Client->>Logout Endpoint: GET /auth/logout<br/>+ Bearer token
    Logout Endpoint->>Auth Service: Extract session ID<br/>from token
    Auth Service->>MongoDB: Delete session record<br/>from identity.sessions
    MongoDB->>Auth Service: Deleted
    Auth Service->>Redis: Blacklist session ID<br/>(key: auth:blacklist:session-id)
    Redis->>Auth Service: OK
    Auth Service->>Logout Endpoint: Session deleted
    Logout Endpoint->>Client: { "result": "OK" }
```

After logout:
1. The session is deleted from the database
2. The session ID is added to a Redis blacklist
3. Any subsequent request with that token is rejected by `AuthGuard` with `401 Unauthorized`
4. Both JWT and APT tokens using that session are invalidated

::: tip Already logged out
If the session is already on the blacklist, `logout` returns `"OK"` without error — the operation is idempotent.
:::

## POST /auth/check — Validate a TOTP code

Validates a one-time password (TOTP) against an encrypted secret. Used internally by OTP, 2FA, and account-verify flows. This is **not** a JWT validity check.

Requires a valid Bearer token.

```bash
curl -X POST http://localhost:3010/auth/check \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "secret": "<base64(AES-encrypt(hex-secret))>",
    "token": "482910",
    "period": 30,
    "window": 1
  }'
```

### Fields

| Field | Type | Default | Description |
|---|---|---|---|
| `secret` | `string` | N/A | AES-encrypted, base64-encoded TOTP hex secret |
| `token` | `string` | N/A | 6-digit OTP submitted by the user |
| `period` | `number` | `30` | TOTP window in seconds (RFC 6238) |
| `window` | `number` | `1` | Drift tolerance in periods (allows ±N late/early entries) |

**Response:** `{ "result": "OK" }` on success, `{ "result": "NOK" }` on mismatch.

### TOTP validation process

1. Decrypt the `secret` value using AES
2. Parse as hex-encoded TOTP secret
3. Generate TOTP codes for current time ± `window` periods
4. Check if `token` matches any generated code
5. Return `"OK"` if match, `"NOK"` if not

### 2FA integration example

```typescript
// 1. User has already logged in with username/password (session_token issued)
// 2. User submits their authenticator app code

const verified = await fetch('/auth/check', {
  method: 'POST',
  headers: { 'Authorization': `Bearer ${session_token}` },
  body: JSON.stringify({
    secret: user.mfa_secret,  // AES-encrypted, from database
    token: '482910',
    period: 30,
    window: 1
  })
})
.then(r => r.json())
.then(r => r.result === 'OK');

if (verified) {
  // Issue full access token after 2FA confirmation
} else {
  showError('Invalid code, please try again');
}
```

## Scope Hierarchy & Elevation

Scopes in Wenex follow a hierarchical structure where higher-privilege scopes can satisfy lower-privilege requirements. This is called **scope elevation**.

### Scope Format

All scopes follow the pattern: `{action}:{service}:{collection?}`

```
read:auth                    # Read anything in auth service
read:auth:apts               # Read only APT tokens
write:identity:users         # Write to users in identity service
manage:content               # Manage anything in content service
whole                        # Wildcard — access to everything
```

### Scope Elevation Rules

| Required scope | Satisfied by |
|---|---|
| `read:*` | `write:*` or `manage:*` |
| `write:*` | `manage:*` |
| Any specific scope | `whole` |

**Example:** A token with scope `write:identity:users` can:
- Make `read:identity:users` requests ✅ (elevation: write satisfies read)
- Make `write:identity:users` requests ✅ (exact match)
- Make `manage:identity:users` requests ❌ (cannot downgrade)
- Make `read:auth` requests ❌ (different service)

### Prefix Matching

A token with `read:identity` satisfies requests requiring `read:identity:users`, `read:identity:profiles`, etc. without listing each individually.

Hierarchy example:

```
whole
└── manage:identity
    ├── write:identity
    │   └── read:identity
    │       ├── read:identity:users
    │       ├── read:identity:profiles
    │       └── read:identity:sessions
    └── manage:identity:users
        ├── write:identity:users
        └── read:identity:users
```

### The "whole" Scope

Wildcard — grants access to all operations across all services. Only assign to trusted admin accounts.

## The strict flag and x-api-key

### When to use strict mode

**Strict mode enforces two-factor authentication**: the token holder must present both the token (what you know) and a valid API key (what you have).

**Use strict for:**
- Web backends calling the Platform on behalf of users (prevents direct user API access)
- Third-party integrations requiring shared credentials
- CI/CD deployments using static credentials

**Don't use strict for:**
- Browser SPAs or mobile apps making requests directly (use regular JWT)
- Localhost development (too cumbersome)
- Public read-only operations

### What strict does

When a token is issued with `strict: true`, the `AuthGuard` enforces that every request carrying that token **must also include a valid `x-api-key` header**. Without it the guard returns `403 Forbidden` immediately, before any business logic runs.

**Purpose:** prevents the holder of a `strict` token from calling the Platform API directly. Only a trusted client backend — which possesses the API key — can make Platform calls on behalf of that token.

### Creating strict tokens

```bash
curl -X POST http://localhost:3010/auth/token \
  -H "Content-Type: application/json" \
  -d '{
    "grant_type": "password",
    "client_id": "64a1b2c3d4e5f6a7b8c9d0e1",
    "username": "user@example.com",
    "password": "secret",
    "strict": true
  }'
```

The response includes `"strict": true` in the JWT payload — visible when calling `/auth/verify`.

### Client API Token Header — x-api-key

The `x-api-key` header contains a cryptographically signed proof that your backend possesses a shared secret. Its value is encrypted with AES:

```
x-api-key: base64(AES-encrypt(JSON(ApiToken)))
```

#### ApiToken structure

```typescript
type ApiToken = {
  cid: string;             // must match token.cid
  client_id: string;       // must match token.client_id
  whitelist?: string[];    // optional IP allowlist — caller IP must be included
  expiration_date: Date;   // must be in the future
};
```

| Field | Description |
|---|---|
| `cid` | Client/App ID — must match the JWT's `cid` field exactly |
| `client_id` | OAuth client identifier — must match the JWT's `client_id` field |
| `whitelist` | (Optional) Array of CIDR-notation IPs. Empty array = no restriction |
| `expiration_date` | Absolute date when this API key expires |

#### Generating an API key

Generate once at application startup and reuse.

**Node.js / TypeScript:**

```typescript
import { AES, enc } from 'crypto-js';

const apiToken = {
  cid: jwt.cid,
  client_id: jwt.client_id,
  expiration_date: new Date(Date.now() + 365 * 24 * 60 * 60 * 1000),
  whitelist: ['192.168.1.0/24', '10.0.0.0/8']  // optional
};

const encrypted = AES.encrypt(JSON.stringify(apiToken), process.env.AES_SECRET);
const apiKey = enc.Base64.stringify(encrypted);
```

**Python:**

```python
from cryptography.hazmat.primitives.ciphers import Cipher, algorithms, modes
from cryptography.hazmat.backends import default_backend
import json, base64, os
from datetime import datetime, timedelta

api_token = {
    'cid': jwt_cid,
    'client_id': jwt_client_id,
    'expiration_date': (datetime.utcnow() + timedelta(days=365)).isoformat() + 'Z',
    'whitelist': ['192.168.1.0/24']
}

aes_secret = os.environ['AES_SECRET'].encode()  # 32 bytes for AES-256
iv = os.urandom(16)
cipher = Cipher(algorithms.AES(aes_secret), modes.CBC(iv), backend=default_backend())
encryptor = cipher.encryptor()
plaintext = json.dumps(api_token).encode()
padding_length = 16 - (len(plaintext) % 16)
plaintext += bytes([padding_length] * padding_length)
ciphertext = encryptor.update(plaintext) + encryptor.finalize()
api_key = base64.b64encode(iv + ciphertext).decode()
```

**Bash / OpenSSL:**

```bash
API_TOKEN_JSON='{"cid":"...","client_id":"...","expiration_date":"2027-01-01T00:00:00Z"}'
API_KEY=$(echo -n "$API_TOKEN_JSON" | openssl enc -aes-256-cbc -base64 -pass env:AES_SECRET)
```

#### Using the x-api-key header

Every request with a strict token **must** include the encrypted API key.

**curl:**

```bash
curl http://localhost:3010/identity/users \
  -H "Authorization: Bearer $JWT" \
  -H "x-api-key: $API_KEY"
```

**Node.js:**

```typescript
async function callPlatform(endpoint: string, options: RequestInit = {}) {
  return fetch(`http://localhost:3010${endpoint}`, {
    ...options,
    headers: {
      'Authorization': `Bearer ${process.env.WENEX_TOKEN}`,
      'x-api-key': process.env.WENEX_API_KEY,
      ...options.headers
    }
  });
}
```

**Python:**

```python
import requests, os

headers = {
    'Authorization': f'Bearer {os.environ["WENEX_TOKEN"]}',
    'x-api-key': os.environ['WENEX_API_KEY']
}
response = requests.get('http://localhost:3010/identity/users', headers=headers)
```

#### IP whitelisting

```typescript
// No restriction — any IP allowed
{ cid: "...", client_id: "...", expiration_date: "2027-06-01", whitelist: [] }

// CIDR whitelist
{
  cid: "...", client_id: "...", expiration_date: "2027-06-01",
  whitelist: ["192.168.1.0/24", "10.0.0.0/8", "203.0.113.42"]
}
```

Requests from outside the whitelist receive `403 Forbidden` with `"message": "ip whitelist validation failed"`.

#### Header placement

```
GET /identity/users HTTP/1.1
Host: localhost:3010
Authorization: Bearer eyJ...
x-api-key: <encrypted-ApiToken>
```

### Validation sequence (`AuthShield.check`)

1. Extract `x-api-key` header from request.
2. Base64-decode and AES-decrypt the value.
3. Parse as `ApiToken` — throw `403 Forbidden` if malformed.
4. Assert `cid` matches `token.cid`.
5. Assert `client_id` matches `token.client_id`.
6. Assert `expiration_date` is in the future.
7. If `whitelist` is non-empty: extract caller IP and check against CIDR ranges.
8. Assert `token.type === "access"` — refresh tokens cannot be used for API calls.
9. All checks passed — allow request to proceed ✅

### Error handling

| Error | Response message | Fix |
|---|---|---|
| Missing `x-api-key` (strict token) | `"strict token must have valid api-key"` | Add `x-api-key` header to every request |
| Invalid/malformed key | `"api-key is not valid"` | Check AES encryption and base64 encoding |
| Expired key | `"api-key is not valid"` | Regenerate key with future `expiration_date` |
| `cid` mismatch | `"api-key is not valid"` | Ensure `ApiToken.cid` matches the JWT's `cid` |
| IP not whitelisted | `"invalid ip"` | Add your IP to `whitelist` or remove whitelisting |

### Behaviour matrix

| Scenario | `token.strict` | `x-api-key` header | Result |
|:---|:---:|:---:|---|
| Backend with valid key, fresh, IP allowed | `true` | ✅ | **Allowed** |
| End user using token directly | `true` | ❌ missing | **403 Forbidden** |
| Expired API key | `true` | ✅ expired | **403 Forbidden** |
| Wrong `cid` in API token | `true` | ❌ mismatch | **403 Forbidden** |
| IP not in whitelist | `true` | ✅ wrong IP | **403 Forbidden** |
| Non-strict token | `false` | any (ignored) | **Allowed** |

### Regular vs strict token comparison

| Aspect | Regular token | Strict token |
|---|---|---|
| **End user can call API directly** | ✅ YES | ❌ NO |
| **Backend can call API** | ✅ YES | ✅ YES |
| **`x-api-key` required** | ❌ NO | ✅ YES |
| **Typical use case** | Browser SPA, mobile app | Web backend, CI/CD |
| **Security level** | Medium (token alone) | High (token + API key) |

### Security best practices

1. **Never expose API keys** — use environment variables, not hardcoded strings
2. **Rotate keys regularly** — regenerate every 90 days
3. **Use IP whitelisting in production** — list only your web/API/worker server IPs
4. **Set appropriate expiration dates** — development: 1–2 years; production: 90–365 days; CI/CD: 30–90 days
5. **Use different keys per environment** — `WENEX_API_KEY_DEV` vs `WENEX_API_KEY_PROD`
6. **Log and monitor usage** — alert on requests from unexpected IPs or repeated failures
7. **Revoke compromised keys immediately** — delete the token, generate a replacement, restart services

## APTs — Auth Personal Tokens

APTs are long-lived, revocable credentials stored in Redis and used for server-to-server authentication, CI/CD pipelines, and AI agents. They carry the same claims as a JWT and are usable everywhere a JWT is accepted.

### APT vs JWT comparison

| Aspect | JWT | APT |
|---|---|---|
| **Storage** | Signed, stateless (no server-side storage) | Redis cache: `auth:apt:<suffix>` |
| **Lifetime** | Short-lived (typically 1 hour) | Long-lived, configurable per token |
| **Revocation** | Not revocable — must wait for expiry | Revocable immediately by deleting the APT |
| **Use case** | Interactive user sessions, OAuth flows | Server-to-server, CI/CD bots, AI agents |
| **Token format** | `eyJ...` (base64 JWT) | `APT-<base62-encoded-id>` |
| **Refresh** | Requires new token request via `/auth/token` | APT tokens don't expire unless deleted |

### Create an APT

```bash
curl -X POST http://localhost:3010/auth/apts \
  -H "Authorization: Bearer $JWT" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "ci-bot",
    "scopes": ["read:identity:users"],
    "subjects": ["ci@example.com"],
    "strict": false,
    "coworkers": ["64a1b2c3d4e5f6a7b8c9d0e1"]
  }'
```

**Response:**

```json
{
  "data": {
    "id": "64a1b2c3d4e5f6a7b8c9d0e3",
    "name": "ci-bot",
    "token": "APT-xxxxxxxxxxxxxxxxxxxxxxxx",
    "scopes": ["read:identity:users"],
    "subjects": ["ci@example.com"],
    "created_at": "2026-05-15T00:00:00.000Z"
  }
}
```

::: warning Store immediately
The `token` field is returned **only at creation**. It cannot be retrieved again — store it securely in a secret manager or environment variable.
:::

### APT fields reference

| Field | Required | Type | Description |
|---|:---:|---|---|
| `name` | ✅ | `string` | Human-readable label (e.g., `"ci-bot"`, `"aws-lambda"`) |
| `scopes` | ✅ | `Scope[]` | OAuth scopes this APT is allowed to use |
| `subjects` | ✅ | `string[]` | ABAC subjects — same format as JWT `subject` field |
| `strict` | | `boolean` | If `true`, requests must also include a valid `x-api-key` header. Default: `false` |
| `expires_at` | | `number` | Unix milliseconds when this APT expires. If omitted, uses server default |
| `coworkers` | | `string[]` | Additional coworker client IDs to add to the APT's `clients[]` field |

### Using an APT

```bash
TOKEN="APT-xxxxxxxxxxxxxxxxxxxxxxxx"

curl http://localhost:3010/identity/users \
  -H "Authorization: Bearer $TOKEN"
```

The platform recognizes the `APT-` prefix, resolves it from Redis, and processes the request identically to a JWT-based request.

### APT verification flow

When a request arrives with `Authorization: Bearer APT-...`:

1. Extract suffix from the token (`APT-<suffix>`)
2. Query Redis at key `auth:apt:<suffix>`
3. If found: decrypt the APT record with AES
4. Convert to JWT claims via the `aptToken()` utility
5. Proceed as if it were a JWT — same scope checking, ABAC authorization, etc.

::: info Redis caching
APTs are encrypted at rest in Redis. If the APT is not found (deleted or expired), the request is rejected with `401 Unauthorized`.
:::

### Revoke an APT

```bash
curl -X DELETE http://localhost:3010/auth/apts/64a1b2c3d4e5f6a7b8c9d0e3 \
  -H "Authorization: Bearer $JWT"
```

This removes the APT from MongoDB and Redis immediately — revocation is **instantaneous**, unlike JWTs which must wait for expiration.

### APT with strict mode

APTs can also enforce strict mode, requiring an `x-api-key` header for two-factor authentication (token + API key):

```bash
curl -X POST http://localhost:3010/auth/apts \
  -H "Authorization: Bearer $JWT" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "aws-lambda-strict",
    "scopes": ["read:identity:users"],
    "subjects": ["lambda@example.com"],
    "strict": true
  }'
```

### Example — CI/CD bot with APT

```bash
# 1. Create the APT (as admin)
curl -X POST http://localhost:3010/auth/apts \
  -H "Authorization: Bearer $ADMIN_JWT" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "github-actions-bot",
    "scopes": ["read:content", "write:content"],
    "subjects": ["ci@github.example.com"]
  }'

# Response includes: "token": "APT-xxxxxxxxxxxxxxxxxxxxxxxx"

# 2. Store in GitHub Actions secret: ${{ secrets.WENEX_TOKEN }}

# 3. Use in CI/CD workflow
export WENEX_TOKEN="APT-xxxxxxxxxxxxxxxxxxxxxxxx"
curl http://localhost:3010/content/notes \
  -H "Authorization: Bearer $WENEX_TOKEN"
```

## POST /auth/can — Check permissions

Checks whether the current token has permission to perform a specific action on a specific resource. Useful for client applications to determine feature availability or UI visibility before attempting an operation.

### Request body

```typescript
{
  action: Action;            // required — e.g., "read", "write", "manage"
  object: Resource;          // required — e.g., "content:notes", "identity:users"
  fields?: string[];         // optional — specific fields to check access for
  filter?: Record<...>;      // optional — query filter context
}
```

### Examples

**Check if token can read users:**

```bash
curl -X POST http://localhost:3010/auth/can \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{ "action": "read", "object": "identity:users" }'
```

**Response — granted:**

```json
{
  "data": {
    "granted": true,
    "policies": [{ "subject": "admin", "action": "read", "object": "identity:users", "field": null, "filter": null }]
  }
}
```

**Check field-level access:**

```bash
curl -X POST http://localhost:3010/auth/can \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{ "action": "write", "object": "identity:users", "fields": ["name", "email", "phone", "password"] }'
```

**Response — some fields denied:**

```json
{
  "data": {
    "granted": false,
    "denied_fields": ["password"],
    "allowed_fields": ["name", "email", "phone"],
    "policies": [{ "subject": "user", "action": "write", "object": "identity:users", "field": ["name", "email", "phone"] }]
  }
}
```

### Pre-flight UI checks

```typescript
const canDelete = await fetch('/auth/can', {
  method: 'POST',
  headers: { 'Authorization': `Bearer ${token}` },
  body: JSON.stringify({ action: 'manage', object: 'content:notes' })
})
.then(r => r.json())
.then(r => r.data.granted);

if (canDelete) showDeleteButton();
else hideDeleteButton();
```

## Client Integration

### SDK (@wenex/sdk)

The official SDK handles token storage and auto-refresh automatically.

```bash
npm install @wenex/sdk
```

```typescript
import { WenexClient } from '@wenex/sdk';

const client = new WenexClient({
  baseURL: 'http://localhost:3010',
  clientId: 'your-app-id'
});

const { access_token } = await client.auth.login({
  username: 'user@example.com',
  password: 'password'
});

// SDK refreshes tokens automatically when they expire
const users = await client.identity.users.find();
```

### JavaScript / TypeScript

```typescript
async function authenticate() {
  const { data } = await fetch('http://localhost:3010/auth/token', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      grant_type: 'password',
      client_id: CLIENT_ID,
      username: 'user@example.com',
      password: 'password'
    })
  }).then(r => r.json());

  return { access_token: data.access_token, refresh_token: data.refresh_token };
}

async function apiCall(endpoint: string, options: RequestInit = {}) {
  let token = getToken();

  let response = await fetch(`http://localhost:3010${endpoint}`, {
    ...options,
    headers: { 'Authorization': `Bearer ${token}`, ...options.headers }
  });

  if (response.status === 401) {
    const { data } = await fetch('http://localhost:3010/auth/token', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({
        grant_type: 'refresh_token',
        client_id: CLIENT_ID,
        refresh_token: getRefreshToken()
      })
    }).then(r => r.json());

    storeToken(data.access_token, data.refresh_token);

    response = await fetch(`http://localhost:3010${endpoint}`, {
      ...options,
      headers: { 'Authorization': `Bearer ${data.access_token}`, ...options.headers }
    });
  }

  return response;
}
```

### Python

```python
import requests

def authenticate(username, password):
    data = requests.post(
        'http://localhost:3010/auth/token',
        json={'grant_type': 'password', 'client_id': 'app-id',
              'username': username, 'password': password}
    ).json()['data']
    return data['access_token'], data['refresh_token']

token, refresh = authenticate('user@example.com', 'password')
session = requests.Session()
session.headers.update({'Authorization': f'Bearer {token}'})

users = session.get('http://localhost:3010/identity/users').json()
```

Auto-refresh session:

```python
class RefreshingSession(requests.Session):
    def __init__(self, client_id, refresh_token_fn):
        super().__init__()
        self.client_id = client_id
        self.refresh_token_fn = refresh_token_fn

    def request(self, *args, **kwargs):
        response = super().request(*args, **kwargs)
        if response.status_code == 401 and self.refresh_token_fn:
            new_token = self.refresh_token_fn()
            self.headers.update({'Authorization': f'Bearer {new_token}'})
            response = super().request(*args, **kwargs)
        return response
```

### Go

```go
func authenticate(username, password string) (string, error) {
    payload, _ := json.Marshal(map[string]string{
        "grant_type": "password",
        "client_id":  "app-id",
        "username":   username,
        "password":   password,
    })

    resp, _ := http.Post(
        "http://localhost:3010/auth/token",
        "application/json",
        bytes.NewReader(payload),
    )

    var result struct {
        Data struct { AccessToken string `json:"access_token"` } `json:"data"`
    }
    json.NewDecoder(resp.Body).Decode(&result)
    return result.Data.AccessToken, nil
}

type AuthenticatedClient struct {
    token string
    http  *http.Client
}

func (c *AuthenticatedClient) Get(path string) (*http.Response, error) {
    req, _ := http.NewRequest("GET", "http://localhost:3010"+path, nil)
    req.Header.Set("Authorization", "Bearer "+c.token)
    return c.http.Do(req)
}
```

### iOS (Swift)

```swift
class AuthService {
    func login(username: String, password: String) async throws {
        struct AuthRequest: Codable {
            let grant_type = "password"
            let client_id: String
            let username: String
            let password: String
        }

        let request = AuthRequest(client_id: "ios-app", username: username, password: password)
        var urlRequest = URLRequest(url: URL(string: "http://localhost:3010/auth/token")!)
        urlRequest.httpMethod = "POST"
        urlRequest.httpBody = try JSONEncoder().encode(request)
        urlRequest.setValue("application/json", forHTTPHeaderField: "Content-Type")

        let (data, _) = try await URLSession.shared.data(for: urlRequest)
        let response = try JSONDecoder().decode(AuthResponse.self, from: data)

        // Store in Keychain — never in UserDefaults
        try KeychainService().store(key: "auth_token", value: response.data.access_token)
    }
}
```

### Android (Kotlin)

```kotlin
interface AuthService {
    @POST("/auth/token")
    suspend fun login(@Body request: AuthRequest): AuthResponse
}

val service = Retrofit.Builder()
    .baseUrl("http://localhost:3010")
    .addConverterFactory(GsonConverterFactory.create())
    .build()
    .create(AuthService::class.java)

val response = service.login(AuthRequest(
    client_id = "android-app",
    username = "user@example.com",
    password = "password"
))

// Store token securely in EncryptedSharedPreferences
val prefs = EncryptedSharedPreferences.create(
    requireContext(), "auth",
    MasterKey.Builder(requireContext()).setKeyScheme(MasterKey.KeyScheme.AES256_GCM).build(),
    EncryptedSharedPreferences.PrefKeyEncryptionScheme.AES256_SIV,
    EncryptedSharedPreferences.PrefValueEncryptionScheme.AES256_GCM
)
prefs.edit().putString("token", response.data.access_token).apply()
```

### CLI / Scripts (APT)

For automation, scripts, and CI/CD — use APTs. They're long-lived and can be revoked immediately.

```bash
# 1. Create an APT (as admin)
curl -X POST http://localhost:3010/auth/apts \
  -H "Authorization: Bearer $ADMIN_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "backup-script",
    "scopes": ["read:content", "write:content"],
    "subjects": ["backup@example.com"]
  }'

# 2. Store the returned token (only shown once)
export WENEX_TOKEN="APT-xxx..."

# 3. Use in scripts
curl http://localhost:3010/content/notes \
  -H "Authorization: Bearer $WENEX_TOKEN"
```

```python
#!/usr/bin/env python3
import os, requests

token = os.environ.get('WENEX_TOKEN')
response = requests.get(
    'http://localhost:3010/identity/users',
    headers={'Authorization': f'Bearer {token}'}
)
print(response.json())
```

### Backend Services

**Node.js / Express:**

```typescript
const TOKEN = process.env.WENEX_SERVICE_TOKEN;

app.get('/sync-users', async (req, res) => {
  const users = await fetch('http://localhost:3010/identity/users', {
    headers: { 'Authorization': `Bearer ${TOKEN}` }
  }).then(r => r.json());
  res.json(users);
});
```

**Django / Python:**

```python
from django.conf import settings
import requests

def sync_users():
    return requests.get(
        'http://localhost:3010/identity/users',
        headers={'Authorization': f'Bearer {settings.WENEX_SERVICE_TOKEN}'}
    ).json()
```

**ASP.NET Core:**

```csharp
services.AddHttpClient<WenexClient>(client => {
    client.BaseAddress = new Uri("http://localhost:3010");
    client.DefaultRequestHeaders.Add("Authorization",
        $"Bearer {configuration["Wenex:Token"]}");
});
```

### Token storage

```typescript
// ✅ httpOnly cookie (most secure for web)
response.cookie('token', token, { httpOnly: true, secure: true, sameSite: 'strict' });

// ✅ Environment variable (CI/CD or backend services)
export WENEX_TOKEN="APT-xxx"

// ❌ localStorage — vulnerable to XSS
localStorage.setItem('token', token);

// ❌ Hardcoded in source
const TOKEN = "eyJ...";
```

### Best practices

1. Use HTTPS only in production
2. Store tokens securely — never in `localStorage` (XSS risk)
3. Handle token expiration gracefully — auto-refresh or re-authenticate
4. Use environment variables — never hardcode tokens
5. Use APTs for CI/CD and long-lived services
6. Request only the scopes you need — apply least-privilege
7. Call `/auth/logout` on user sign-out to invalidate the session server-side
8. Monitor authentication logs — watch for failed attempts or unusual IPs

## Troubleshooting

### 401 Unauthorized: Missing or invalid token

```json
{ "statusCode": 401, "message": "authorization token is required", "error": "Unauthorized" }
```

**Causes:** Missing header, malformed token, or expired token.

```bash
# Inspect token claims and check expiration
curl http://localhost:3010/auth/verify -H "Authorization: Bearer $TOKEN"

# Refresh if expired
curl -X POST http://localhost:3010/auth/token \
  -d '{ "grant_type": "refresh_token", "refresh_token": "..." }'
```

### 401 Unauthorized: Session blacklisted

**Cause:** Token's session was deleted (logged out). **Fix:** Get a new token via `/auth/token`.

### 403 Forbidden: Insufficient scope

**Cause:** Token doesn't have the required scope. **Fix:** Include the required scope when requesting a token: `"scopes": ["read:identity:users"]`.

### 403 Forbidden: Strict token missing API key

```json
{ "statusCode": 403, "message": "strict token must have valid api-key", "error": "Forbidden" }
```

**Fix:** Add `x-api-key: $API_KEY` header to every request.

### 403 Forbidden: Action denied by policy

**Cause:** Token has the right scope but no ABAC grant for this action. **Fix:** Ask admin to create a grant. Use `POST /auth/can` to diagnose.

### 429 Too Many Requests: Rate limited

**Cause:** Too many failed login attempts. **Fix:** Wait 15 minutes, verify your credentials.

## Request headers reference

| Header | Required | Example | Description |
|---|---|---|---|
| `Authorization` | Yes* | `Bearer eyJ…` | JWT or APT. *Omit only on `@IsPublic()` routes |
| `Content-Type` | POST / PATCH | `application/json` | Body encoding |
| `x-api-key` | Conditional | `<encrypted>` | Required when `token.strict === true` |
| `x-domain` | No | `my-tenant.com` | Override the tenant domain |
| `x-request-id` | No | `uuid-v4` | Trace ID — auto-generated if absent |
| `x-zone` | No | `own,share` | Ownership zone filter — default `own` |

## Error responses

| Status | Meaning |
|---|---|
| `401 Unauthorized` | Missing, expired, or invalid token |
| `403 Forbidden` | Token valid but `strict` requires `x-api-key`, or key is invalid |
| `429 Too Many Requests` | Rate limit exceeded |

## Summary: All Flows at a Glance

| Flow | Endpoint | Purpose | Input | Output |
|---|---|---|---|---|
| **Token** | `POST /auth/token` | Exchange credentials for JWT | Grant type + credentials | `access_token` + `refresh_token` |
| **Verify** | `GET /auth/verify` | Inspect token claims | Bearer token | All JWT claims |
| **Logout** | `GET /auth/logout` | Invalidate session | Bearer token | `{ "result": "OK" }` |
| **Can** | `POST /auth/can` | Check permission before action | action + object | `{ granted: boolean, policies: [...] }` |
| **Check** | `POST /auth/check` | Validate TOTP code | secret + 6-digit code | `{ "result": "OK" \| "NOK" }` |

## See Also

- [Authorization](/api/authorization) — ABAC, grants, guards, and the `AuthorityInterceptor`
- [Access Control](/getting-started/overview/key-concepts/access-control) — Core ABAC concepts
