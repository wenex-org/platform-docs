# Filtering, Pagination & Populate

All `find` and `count` endpoints accept a structured filter system. This document describes every component of that system and shows how to use it via REST query parameters, REST body, and GraphQL variables.

## Filter Object Structure

The full `FilterDto` has four optional fields:

```typescript
interface FilterDto<T> {
  query:      Query<T>;          // Required for find/count — MongoDB query expression
  populate?:  PopulateDto[];     // Populate related documents (join)
  projection?: Projection<T>;   // Field inclusion / exclusion
  pagination?: PaginationDto<T>; // limit, skip, sort
}
```

For single-document lookups (`findOne`, `findById`) the `FilterOneDto` is used — it has the same fields minus `pagination`.

For `count` and `updateBulk` a `QueryFilterDto` is used — it has `query` only.

## query — MongoDB Query Expression

`query` accepts any valid MongoDB query object. The platform passes it directly to the Mongoose `find()` / `findOne()` call.

### Basic examples

```jsonc
// Match all documents
{ "query": {} }

// Exact match
{ "query": { "status": "active" } }

// Multiple conditions (implicit AND)
{ "query": { "status": "active", "type": "premium" } }

// Regex (case-insensitive email domain search)
{ "query": { "email": { "$regex": "@example\\.com$", "$options": "i" } } }

// Comparison operators
{ "query": { "amount": { "$gte": 100, "$lte": 1000 } } }

// OR condition
{ "query": { "$or": [{ "status": "active" }, { "status": "pending" }] } }

// Nested field
{ "query": { "address.city": "Tehran" } }

// Array contains value
{ "query": { "tags": { "$in": ["vip", "early-adopter"] } } }

// Field exists check
{ "query": { "phone": { "$exists": true } } }
```

### Passing query as a REST query parameter

URL-encode the JSON string:

```bash
# Using --data-urlencode (curl handles encoding)
curl "$BASE/identity/users" \
  --get \
  --data-urlencode 'query={"status":"active"}' \
  -H "Authorization: Bearer $TOKEN"
```

### Passing the full filter as REST query parameters

Each field of the filter can be passed as a separate query parameter:

```bash
curl "$BASE/identity/users" \
  --get \
  --data-urlencode 'query={"status":"active"}' \
  --data-urlencode 'pagination={"limit":20,"skip":0,"sort":{"created_at":-1}}' \
  --data-urlencode 'projection={"username":1,"email":1}' \
  -H "Authorization: Bearer $TOKEN"
```

## pagination — Limit, Skip, Sort

Controls result set size and ordering.

```typescript
interface PaginationDto<T> {
  limit?: number;   // Max documents per page (has a server-configured maximum)
  skip?:  number;   // Offset (0-based)
  sort?:  { [field in keyof T]: SortOrder }; // MongoDB sort expression
}
```

### Sort values

| Value | Meaning |
|---|---|
| `1` or `"asc"` | Ascending |
| `-1` or `"desc"` | Descending |
| `{ "$meta": "textScore" }` | Relevance score (full-text search) |

### Examples

```jsonc
// Last 10 created
{ "pagination": { "limit": 10, "skip": 0, "sort": { "created_at": -1 } } }

// Page 3 of 20-per-page results
{ "pagination": { "limit": 20, "skip": 40 } }

// Sort by multiple fields
{ "pagination": { "sort": { "priority": -1, "created_at": 1 } } }
```

## projection — Field Inclusion / Exclusion

Standard MongoDB projection. Use `1` to include fields, `0` to exclude.

```jsonc
// Include only specific fields
{ "projection": { "username": 1, "email": 1, "created_at": 1 } }

// Exclude sensitive fields (all others included)
{ "projection": { "password": 0, "secret": 0 } }
```

> You cannot mix inclusions and exclusions in the same projection (MongoDB restriction), except for `_id`.

## populate — Populate Related Documents

Populate resolves MongoId references to embedded documents. Each entry in the `populate` array specifies one relationship to resolve.

```typescript
interface PopulateDto {
  path:     string;            // Field path to populate (e.g. "owner", "identity", "relations")
  model?:   string;            // Mongoose model of the target collection —
                               // REQUIRED for the polymorphic `identity` / `relations` fields
  select?:  Projection;        // Fields to include / exclude from the related document
  match?:   Query;             // Filter applied to the populated documents
  options?: PaginationDto;     // limit / skip / sort for the populated documents
}
```

::: warning `model` is required for polymorphic relations
`identity` (single) and `relations` (array) are polymorphic — they can point at any collection — so a `populate` entry for either **must** include `model` (the target collection's Mongoose model name, e.g. `IdentityUser`, `ConjointMember`, `FinancialWallet`). Fixed-target fields like `owner` do not need it. Omitting `model` on a relational field is rejected server-side (`population "model" is required`).
:::

### Examples

```jsonc
// Populate the owner field (returns the User document inline)
{
  "populate": [
    { "path": "owner", "select": ["username", "email"] }
  ]
}

// Multiple populates
{
  "populate": [
    { "path": "owner" },
    { "path": "client" }
  ]
}

// Polymorphic fields — `model` selects the target collection
{
  "populate": [
    { "path": "identity",  "model": "IdentityUser" },
    { "path": "relations", "model": "ConjointMember", "select": ["id", "username"] }
  ]
}
```

### REST example with populate

```bash
curl "$BASE/identity/users" \
  --get \
  --data-urlencode 'query={}' \
  --data-urlencode 'populate=[{"path":"owner","select":["username","email"]}]' \
  -H "Authorization: Bearer $TOKEN"
```

### GraphQL example with populate

```graphql
query {
  findIdentityUser(filter: {
    query: {},
    populate: [{ path: "owner", select: ["username", "email"] }]
  }) {
    data {
      id
      username
      owner {
        id
        username
        email
      }
    }
  }
}
```

## zone — Ownership Filtering

The `zone` parameter is a query parameter (not part of the filter body) that applies automatic ownership scoping via `AuthorityInterceptor`.

| Value | Behavior |
|---|---|
| `own` | Documents where `owner` equals authenticated user |
| `share` | Documents where authenticated user is in `shares[]` |
| `group` | Documents where authenticated user's domain/email matches `groups[]` |
| `client` | Documents belonging to the OAuth `client_id` in the token |

Zones can be combined with commas. Note the combination is **not** a plain union: `own`/`share` are OR-ed together, while `group` and `client` are each AND-ed as additional constraints. See [Access Control → Zone Combination Logic](../getting-started/overview/key-concepts/access-control.md#zone-combination-logic) for the exact semantics.

```bash
# My own documents
curl "$BASE/identity/users?query={}&zone=own" \
  -H "Authorization: Bearer $TOKEN"

# My own + shared with me  (own OR share)
curl "$BASE/identity/users?query={}&zone=own,share" \
  -H "Authorization: Bearer $TOKEN"

# (own OR share) AND group AND client
curl "$BASE/identity/users?query={}&zone=own,share,group,client" \
  -H "Authorization: Bearer $TOKEN"
```

## Complete Filter Example

REST request with all filter options:

```bash
curl "$BASE/financial/transactions" \
  --get \
  --data-urlencode 'query={"status":"completed","amount":{"$gte":100}}' \
  --data-urlencode 'populate=[{"path":"account","select":["name","balance"]}]' \
  --data-urlencode 'projection={"amount":1,"status":1,"created_at":1}' \
  --data-urlencode 'pagination={"limit":20,"skip":0,"sort":{"created_at":-1}}' \
  -H "Authorization: Bearer $TOKEN"
```

Equivalent GraphQL:

```graphql
query GetTransactions($filter: FilterDto!) {
  findFinancialTransaction(filter: $filter) {
    count
    data {
      id
      amount
      status
      created_at
      account {
        id
        name
        balance
      }
    }
  }
}
```

Variables:

```json
{
  "filter": {
    "query": { "status": "completed", "amount": { "$gte": 100 } },
    "populate": [{ "path": "account", "select": ["name", "balance"] }],
    "projection": { "amount": 1, "status": 1, "created_at": 1 },
    "pagination": { "limit": 20, "skip": 0, "sort": { "created_at": -1 } }
  }
}
```

## Filter for Count and Update Bulk

`count` and `updateBulk` accept only `QueryFilterDto`, which contains just `query`:

```bash
# Count completed transactions
curl "$BASE/financial/transactions/count" \
  --get \
  --data-urlencode 'query={"status":"completed"}' \
  -H "Authorization: Bearer $TOKEN"
```

For `PATCH /bulk`, pass the query as a query parameter and the update fields in the JSON body:

```bash
curl -X PATCH "$BASE/financial/transactions/bulk" \
  --get \
  --data-urlencode 'query={"status":"pending"}' \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{ "status": "failed" }'
```

## ETag Caching

Read endpoints emit `ETag` headers. Clients can use conditional requests to avoid re-downloading unchanged data:

```bash
# First request — store the ETag
curl -I "$BASE/identity/users?query={}" \
  -H "Authorization: Bearer $TOKEN"
# ETag: "abc123"

# Conditional request
curl "$BASE/identity/users?query={}" \
  -H "Authorization: Bearer $TOKEN" \
  -H 'If-None-Match: "abc123"'
# → 304 Not Modified (if data hasn't changed)
```
