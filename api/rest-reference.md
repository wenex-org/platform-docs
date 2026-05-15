# REST API Reference

The gateway exposes a uniform REST interface for all 15 domain services. Every collection shares the same 11-endpoint pattern. This document covers the pattern, all available collections, and curl examples using the `identity/users` collection.

**Base URL:** `http://localhost:3010`

All examples assume:
```bash
export TOKEN="eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9..."
export BASE="http://localhost:3010"
```

---

## Standard Endpoint Pattern

Every collection at `/{service}/{collection}` provides these endpoints:

| Method | Path | Scope | Description |
|---|---|---|---|
| `GET` | `/{service}/{collection}/count` | `read:` | Count matching documents |
| `POST` | `/{service}/{collection}` | `write:` | Create one document |
| `POST` | `/{service}/{collection}/bulk` | `write:` | Create many documents |
| `GET` | `/{service}/{collection}` | `read:` | Find with filters and pagination |
| `GET` | `/{service}/{collection}/cursor` | `read:` | Stream results via SSE |
| `GET` | `/{service}/{collection}/:id` | `read:` | Find one by ID |
| `PATCH` | `/{service}/{collection}/:id` | `write:` | Update one by ID |
| `PATCH` | `/{service}/{collection}/bulk` | `manage:` | Update many matching filter |
| `DELETE` | `/{service}/{collection}/:id` | `write:` | Soft-delete by ID |
| `PUT` | `/{service}/{collection}/:id/restore` | `write:` | Restore soft-deleted document |
| `DELETE` | `/{service}/{collection}/:id/destroy` | `manage:` | Hard-delete (permanent) |

> **Scope notation:** `read:` = `read:{service}:{collection}`, e.g. `read:identity:users`

---

## Detailed Endpoint Descriptions

### GET /count — Count Documents

Returns the total number of documents matching the query filter.

```bash
# Count all users
curl "$BASE/identity/users/count?query={}" \
  -H "Authorization: Bearer $TOKEN"

# Count users with a specific email domain
curl "$BASE/identity/users/count" \
  --get \
  --data-urlencode 'query={"email":{"$regex":"@example.com"}}' \
  -H "Authorization: Bearer $TOKEN"
```

**Response:**

```json
{ "total": 42 }
```

---

### POST / — Create One Document

```bash
curl -X POST "$BASE/identity/users" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "username": "jdoe",
    "email": "jdoe@example.com",
    "password": "Str0ng!Pass",
    "name": "John Doe"
  }'
```

**Response:**

```json
{
  "data": {
    "id": "64a1b2c3d4e5f6a7b8c9d0e1",
    "username": "jdoe",
    "email": "jdoe@example.com",
    "name": "John Doe",
    "created_at": "2026-05-15T10:00:00.000Z",
    "updated_at": "2026-05-15T10:00:00.000Z"
  }
}
```

> **Note:** Computed fields (`id`, `created_at`, `updated_at`, `created_by`, etc.) are platform-managed and must not be sent in the request body.

---

### POST /bulk — Create Many Documents

```bash
curl -X POST "$BASE/identity/users/bulk" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "items": [
      { "username": "alice", "email": "alice@example.com" },
      { "username": "bob",   "email": "bob@example.com" }
    ]
  }'
```

**Response:**

```json
{
  "data": [
    { "id": "64a1b2c3...", "username": "alice", ... },
    { "id": "64a1b2c4...", "username": "bob",   ... }
  ],
  "count": 2
}
```

---

### GET / — Find with Filters

Accepts filter, populate, projection, and pagination as query parameters. See [Filtering & Pagination](./filtering.md) for the complete reference.

```bash
# Basic: find all (empty query)
curl "$BASE/identity/users?query={}" \
  -H "Authorization: Bearer $TOKEN"

# With pagination
curl "$BASE/identity/users" \
  --get \
  --data-urlencode 'query={}' \
  --data-urlencode 'pagination={"limit":10,"skip":0,"sort":{"created_at":-1}}' \
  -H "Authorization: Bearer $TOKEN"

# With field projection (include only username and email)
curl "$BASE/identity/users" \
  --get \
  --data-urlencode 'query={}' \
  --data-urlencode 'projection={"username":1,"email":1}' \
  -H "Authorization: Bearer $TOKEN"

# Zone filter: only my own documents
curl "$BASE/identity/users?query={}&zone=own" \
  -H "Authorization: Bearer $TOKEN"
```

**Response:**

```json
{
  "data": [
    { "id": "64a1b2c3...", "username": "alice", ... },
    { "id": "64a1b2c4...", "username": "bob",   ... }
  ],
  "count": 2
}
```

---

### GET /cursor — Stream via SSE

Returns a chunked Server-Sent Events stream. See [Streaming](./streaming.md).

```bash
curl -N "$BASE/identity/users/cursor?query={}" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Accept: text/event-stream"
```

---

### GET /:id — Find One by ID

```bash
curl "$BASE/identity/users/64a1b2c3d4e5f6a7b8c9d0e1" \
  -H "Authorization: Bearer $TOKEN"
```

Optionally pass a `ref` query parameter to look up by the `ref` string field instead:

```bash
curl "$BASE/identity/users/my-ref-slug?ref=true" \
  -H "Authorization: Bearer $TOKEN"
```

**Response:**

```json
{
  "data": {
    "id": "64a1b2c3d4e5f6a7b8c9d0e1",
    "username": "jdoe",
    "email": "jdoe@example.com",
    "created_at": "2026-05-15T10:00:00.000Z"
  }
}
```

---

### PATCH /:id — Update One by ID

Only include fields you want to change. Omitted fields are left unchanged.

```bash
curl -X PATCH "$BASE/identity/users/64a1b2c3d4e5f6a7b8c9d0e1" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{ "name": "Jonathan Doe" }'
```

**Response:** same shape as `GET /:id` with updated fields.

---

### PATCH /bulk — Update Many Documents

Requires `manage:` scope. Updates all documents matching the query filter.

```bash
curl -X PATCH "$BASE/identity/users/bulk" \
  --get \
  --data-urlencode 'query={"status":"inactive"}' \
  -X PATCH \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{ "archived": true }'
```

> For `PATCH /bulk`, pass the filter as query params and the update body as JSON.

**Response:**

```json
{ "total": 5 }
```

---

### DELETE /:id — Soft Delete

Sets `deleted_at` on the document. The document is hidden from normal queries but not permanently removed. Requires `write:` scope.

```bash
curl -X DELETE "$BASE/identity/users/64a1b2c3d4e5f6a7b8c9d0e1" \
  -H "Authorization: Bearer $TOKEN"
```

**Response:** the soft-deleted document with `deleted_at` populated.

---

### PUT /:id/restore — Restore a Soft-Deleted Document

Clears `deleted_at`, making the document visible again. Requires `write:` scope.

```bash
curl -X PUT "$BASE/identity/users/64a1b2c3d4e5f6a7b8c9d0e1/restore" \
  -H "Authorization: Bearer $TOKEN"
```

---

### DELETE /:id/destroy — Hard Delete (Permanent)

Permanently removes the document from MongoDB. **Irreversible.** Requires `manage:` scope.

```bash
curl -X DELETE "$BASE/identity/users/64a1b2c3d4e5f6a7b8c9d0e1/destroy" \
  -H "Authorization: Bearer $TOKEN"
```

---

## All Collections

### Auth Service (`/auth`)

| Collection | Base path | Description |
|---|---|---|
| Auths | `/auth` | Authentication, token, logout |
| APTs | `/auth/apts` | Auth Personal Tokens (API keys) |
| Grants | `/auth/grants` | OAuth permission grants |

> `/auth` is special: it provides token/verify/logout/check/can instead of the standard CRUD pattern. See [Authentication](./authentication.md).

### Identity Service (`/identity`)

| Collection | Base path | Description |
|---|---|---|
| Users | `/identity/users` | User accounts |
| Profiles | `/identity/profiles` | Extended user profile data |
| Sessions | `/identity/sessions` | Active sessions |

### Financial Service (`/financial`)

| Collection | Base path | Description |
|---|---|---|
| Accounts | `/financial/accounts` | Financial accounts |
| Wallets | `/financial/wallets` | Wallets per account |
| Invoices | `/financial/invoices` | Billing invoices |
| Transactions | `/financial/transactions` | Payment transactions |
| Currencies | `/financial/currencies` | Supported currencies |

### Career Service (`/career`)

| Collection | Base path | Description |
|---|---|---|
| Businesses | `/career/businesses` | Business entities |
| Branches | `/career/branches` | Business branches |
| Employees | `/career/employees` | Employee records |
| Products | `/career/products` | Product catalog |
| Services | `/career/services` | Offered services |
| Stocks | `/career/stocks` | Inventory |
| Stores | `/career/stores` | Store locations |
| Customers | `/career/customers` | Customer records |

### Domain Service (`/domain`)

| Collection | Base path | Description |
|---|---|---|
| Apps | `/domain/apps` | OAuth applications |
| Clients | `/domain/clients` | OAuth clients |

### Essential Service (`/essential`)

| Collection | Base path | Description |
|---|---|---|
| Sagas | `/essential/sagas` | Distributed saga orchestrations |
| Saga Stages | `/essential/saga-stages` | Individual saga steps |

### Context Service (`/context`)

| Collection | Base path | Description |
|---|---|---|
| Configs | `/context/configs` | Application configurations |
| Settings | `/context/settings` | User/tenant settings |

### General Service (`/general`)

| Collection | Base path | Description |
|---|---|---|
| Activities | `/general/activities` | Activity log entries |
| Artifacts | `/general/artifacts` | Generic binary artifacts |
| Comments | `/general/comments` | Comment threads |
| Events | `/general/events` | Calendar/system events |
| Workflows | `/general/workflows` | Process workflow definitions |

### Special Service (`/special`)

| Collection | Base path | Description |
|---|---|---|
| Files | `/special/files` | File uploads (public) |
| Stats | `/special/stats` | Aggregated statistics |

### Touch Service (`/touch`)

| Collection | Base path | Description |
|---|---|---|
| Emails | `/touch/emails` | Outbound email records |
| Notices | `/touch/notices` | In-app notifications |
| Pushes | `/touch/pushes` | Push notification records |
| SMS | `/touch/sms` | SMS records |
| Histories | `/touch/histories` | Notification delivery history |

### Content Service (`/content`)

| Collection | Base path | Description |
|---|---|---|
| Notes | `/content/notes` | Notes / rich-text documents |
| Posts | `/content/posts` | Published posts / articles |
| Tickets | `/content/tickets` | Support tickets |

### Logistic Service (`/logistic`)

| Collection | Base path | Description |
|---|---|---|
| Locations | `/logistic/locations` | Geographic locations |
| Drivers | `/logistic/drivers` | Driver profiles |
| Vehicles | `/logistic/vehicles` | Vehicle registry |
| Travels | `/logistic/travels` | Trip / route records |
| Cargoes | `/logistic/cargoes` | Cargo manifests |

### Conjoint Service (`/conjoint`)

| Collection | Base path | Description |
|---|---|---|
| Accounts | `/conjoint/accounts` | Messaging accounts |
| Channels | `/conjoint/channels` | Communication channels |
| Contacts | `/conjoint/contacts` | Contact entries |
| Members | `/conjoint/members` | Channel membership |
| Messages | `/conjoint/messages` | Individual messages |

### Thing Service (`/thing`)

| Collection | Base path | Description |
|---|---|---|
| Devices | `/thing/devices` | IoT device registry |
| Sensors | `/thing/sensors` | Sensor definitions |
| Metrics | `/thing/metrics` | Sensor metric readings |

---

## Response Envelope

All responses are wrapped in a consistent envelope:

### Single document

```json
{
  "data": { ...document fields... }
}
```

### Collection

```json
{
  "data": [ ...documents... ],
  "count": 42
}
```

### Count / total

```json
{ "total": 42 }
```

### Boolean result

```json
{ "result": true }
```

---

## Common Fields on Every Document

These fields are platform-managed and present on every entity returned by the API:

| Field | Type | Description |
|---|---|---|
| `id` | string | MongoDB ObjectId (as string) |
| `ref` | string | Optional human-readable unique reference |
| `owner` | string | MongoId of the owning user |
| `shares` | string[] | MongoIds of users with shared access |
| `groups` | string[] | Email/FQDN list for group access |
| `clients` | string[] | MongoIds of OAuth clients |
| `created_at` | ISO 8601 | Creation timestamp |
| `updated_at` | ISO 8601 | Last update timestamp |
| `deleted_at` | ISO 8601 \| null | Soft-delete timestamp |
| `version` | number | Optimistic concurrency version |
| `tags` | string[] | Free-form tags |
| `props` | object | Free-form extra properties |

---

## Swagger UI

The full interactive API documentation with request/response schemas is available at:

```
http://localhost:3010/api
```

The raw OpenAPI JSON spec is at:

```
http://localhost:3010/api-json
```
