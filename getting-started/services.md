# Services

Wenex Platform consists of 15 domain microservices. All services are NestJS applications that expose a REST API and a gRPC server.

## Architecture Summary

```mermaid
graph LR
    subgraph Core Services
        AUTH["auth<br/>:3020/:5020"]
        DOM["domain<br/>:3030/:5030"]
        CTX["context<br/>:3040/:5040"]
        ESS["essential<br/>:3050/:5050"]
        ID["identity<br/>:3080/:5080"]
    end

    subgraph Business Services
        FIN["financial<br/>:3060/:5060"]
        CAR["career<br/>:3140/:5140"]
        SPE["special<br/>:3090/:5090"]
        TCH["touch<br/>:3100/:5100"]
        CNT["content<br/>:3110/:5110"]
        LOG["logistic<br/>:3120/:5120"]
        CON["conjoint<br/>:3130/:5130"]
        GEN["general<br/>:3070/:5070"]
        THG["thing<br/>:3150/:5150"]
    end

    GW[Gateway :3010] --> Core Services
    GW --> Business Services
    Core Services --> Kafka[(Kafka)]
    Business Services --> Kafka
```

## Standard Service Internals

Every microservice follows the same internal structure:

```
apps/services/<name>/src/
├── main.ts                    # Starts REST listener + gRPC server
├── app.module.ts              # Root module: DB connection, imported modules
├── app.service.ts             # Health checks, initialization
├── modules/
│   └── <entity>/
│       ├── <entity>.module.ts
│       ├── <entity>.controller.ts   # REST (also consumed by gateway via gRPC)
│       ├── <entity>.service.ts      # Business logic + gRPC handler
│       ├── <entity>.repository.ts   # Typegoose (MongoDB) queries
│       ├── <entity>.schema.ts       # Mongoose schema definition
│       └── dto/
│           ├── create-<entity>.dto.ts
│           ├── update-<entity>.dto.ts
│           └── <entity>.serializer.ts
└── protobuf/                  # Generated gRPC TypeScript stubs
```

## Auth

**Port:** REST `:3020` · gRPC `:5020`

Handles all authentication flows, token management, personal API keys, and OAuth permission grants. Unlike every other service, `auth` does not follow the standard 14-operation CRUD pattern for its primary endpoint.

### Collections

| Collection | Path | Purpose |
| --- | --- | --- |
| -- | `/auth` | Token issuance, verification, logout, health check, and ABAC policy evaluation |
| APTs | `/auth/apts` | Long-lived Auth Personal Tokens (API keys) |
| Grants | `/auth/grants` | OAuth permission grants |

### `auth` — Token Endpoint

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

### `auth/apts` — Auth Personal Tokens

Long-lived API keys scoped to a set of OAuth scopes. Use for service-to-service authentication or CI/CD automation.

#### Required Create Fields

| Field | Required | Type | Description |
| --- | :---: | --- | --- |
| `scopes` | ✅ | `string[]` | OAuth scopes this APT is authorized to use |

#### Key Behaviors

- APTs are revocable — delete the record to invalidate immediately.
- Always create APTs with the minimum required scopes.
- APTs appear in `Authorization: Bearer <apt>` headers, the same as JWTs.

### `auth/grants` — Permission Grants

OAuth permission grants define what actions a subject (user or role) may perform on a resource within a domain.

#### Required Create Fields

| Field | Required | Type | Description |
| --- | :---: | --- | --- |
| `action` | ✅ | `Action` enum | `read`, `write`, `manage` |
| `subject` | ✅ | string | Subject identifier (e.g. `guest@example.com`) |
| `resource` | ✅ | string | Resource identifier (e.g. `identity:users`) |
| `domain` | ✅ | string | FQDN of the tenant domain |

#### Key Behaviors

- Grants are evaluated by `PolicyGuard` on every authenticated request.
- Subject format: `{username}@{domain}` — the domain suffix is required in grants but **not** stored in `identity/users.subjects`.
- Use `POST /auth/can` to test whether a subject has a specific grant before making changes.

## Domain

**Port:** REST `:3030` · gRPC `:5030`

Manages tenant domains, OAuth application definitions, and client registrations. Every token and document is scoped to a client context defined in this service.

### Collections

| Collection | Path | Purpose |
| --- | --- | --- |
| Apps | `/domain/apps` | OAuth application definitions under a client |
| Clients | `/domain/clients` | Top-level tenant / OAuth client records |

### `domain/apps`

An application registered under a client. Apps represent runtime surfaces such as web, mobile, desktop, or application contexts.

#### Required Create Fields

| Field | Required | Type | Description |
| --- | :---: | --- | --- |
| `type` | ✅ | `AppType` | `WEB`, `MOBILE`, `DESKTOP`, `APPLICATION` |
| `cid` | ✅ | MongoId | Parent `domain/clients` ID |
| `status` | ✅ | `Status` | `ACTIVE`, `INACTIVE` |

#### Optional Fields

| Field | Type | Description |
| --- | --- | --- |
| `name` | string | Display name |
| `url` | string | App URL |
| `scopes` | string[] | App-allowed scopes — must be a subset of the parent client's scopes |
| `grant_types` | `GrantType[]` | `password`, `refresh_token`, `client_credential`, `authorization_code` |
| `access_token_ttl` | number | Access token TTL in seconds |
| `refresh_token_ttl` | number | Refresh token TTL in seconds |
| `change_logs` | object[] | Nested version / changelog entries |

#### Key Behaviors

- An app cannot expose scopes the parent client does not have.
- Changing `scopes` or TTLs affects downstream token issuance for that app.

### `domain/clients`

The top-level tenant unit. Every token, ABAC scope, and Coworkers relationship originates here.

#### Required Create Fields

| Field | Required | Type | Description |
| --- | :---: | --- | --- |
| `name` | ✅ | string | Client display name |
| `plan` | ✅ | `ClientPlan` | `ALUMINUM`, `GOLD`, `PLATINUM` |
| `status` | ✅ | `Status` | `ACTIVE`, `INACTIVE` |
| `grant_types` | ✅ | `GrantType[]` | Allowed OAuth grant types |

#### Optional Fields

| Field | Type | Description |
| --- | --- | --- |
| `scopes` | string[] | Available OAuth scopes for this client |
| `coworkers` | MongoId[] | Other client IDs in the same Coworkers space |
| `whitelist` | string[] | Allowed IP addresses |
| `access_token_ttl` | number | Access token TTL in seconds |
| `refresh_token_ttl` | number | Refresh token TTL in seconds |
| `domains` | object[] | FQDN entries used for subject-domain resolution |
| `services` | object[] | External integration configurations (SMS, push, email providers) |

#### Platform-Managed Fields (read-only)

| Field | Description |
| --- | --- |
| `api_key` | Platform-generated API key |
| `client_id` | Platform-generated OAuth client ID |
| `client_secret` | Platform-generated OAuth client secret |
| `expiration_date` | Platform-managed expiry |

#### Key Behaviors

- Changes to `domains` affect subject-domain resolution for grants — confirm before updating.
- Changes to `coworkers` affect cross-client data visibility immediately.
- `client_secret` is sensitive — never expose it unnecessarily.

## Context

**Port:** REST `:3040` · gRPC `:5040`

Stores application-wide and per-user configuration data. The two collections serve different scopes: `configs` for entity-scoped platform behavior switches, `settings` for user-scoped preferences.

### Collections

| Collection | Path | Purpose |
| --- | --- | --- |
| Configs | `/context/configs` | Entity-scoped configuration (RBAC, CQRS, feature flags) |
| Settings | `/context/settings` | User-scoped preference storage |

### `context/configs`

Stores configuration entries scoped to an entity ID — a client, app, or user. This is also the registry for all CQRS webhook endpoints.

#### Required Create Fields

| Field | Required | Type | Description |
| --- | :---: | --- | --- |
| `key` | ✅ | `ConfigKey` | Controlled enum key |
| `eid` | ✅ | MongoId | Target entity ID (`uid`, `aid`, or `cid`) |

#### Optional Fields

| Field | Type | Description |
| --- | --- | --- |
| `value` | JSON | Free-form JSON value |
| `status` | `Status` | `ACTIVE` or `INACTIVE` |

#### Known `ConfigKey` Values

| Key | Purpose |
| --- | --- |
| `RBAC` | Role-based access control rules for the target entity |
| `CQRS` | CQRS webhook registration — `value.webhook` is the target URL |
| Collection keys | e.g. `auth/apts`, `identity/users`, `financial/wallets` — per-collection config |

#### CQRS Registration Example

```typescript
await platform.context.configs.create({
  key: 'CQRS',
  eid: '<client_id>',
  value: { webhook: 'http://my-client.com/cqrs' },
})
```

> Changing `RBAC` config affects permission resolution platform-wide for the target entity. Always confirm before updating.

### `context/settings`

Stores typed user preference records.

#### Required Create Fields

| Field | Required | Type | Description |
| --- | :---: | --- | --- |
| `key` | ✅ | string | Setting name (uppercase + underscore pattern) |
| `type` | ✅ | `ValueType` | Declared value type |

#### Optional Fields

| Field | Type | Description |
| --- | --- | --- |
| `value` | JSON | Value matching the declared `type` |
| `status` | `Status` | `ACTIVE` or `INACTIVE` |

#### `ValueType` Values

`NULL`, `ARRAY`, `OBJECT`, `STRING`, `NUMBER`, `BOOLEAN`

#### Key Behaviors

- Use `find_one` before creating a new setting — prefer updating an existing record over creating duplicates.
- Use `?zone=own,client` for current-user preference lookups.

## Essential

**Port:** REST `:3050` · gRPC `:5050`

Manages distributed saga transactions across multiple services. Sagas coordinate multi-step operations (e.g. a business creation that touches `logistic/locations`, `financial/accounts`, and `context/configs` atomically). Workers use PostgreSQL-backed saga stages for durability and crash recovery.

### Collections

| Collection | Path | Purpose |
| --- | --- | --- |
| Sagas | `/essential/sagas` | Top-level orchestration record |
| Saga Stages | `/essential/saga-stages` | Per-step execution history |

### `essential/sagas`

Tracks the top-level orchestration unit for a long-running or distributed operation.

#### Starting a Saga (Recommended)

Use the platform REST `start` endpoint — not the raw CRUD create:

```
POST /essential/sagas/start
{ "ttl": 3600 }
```

The Platform manages `job`, `state`, and `session` automatically. This is the correct path for user-facing saga orchestration.

#### Raw Create Fields (Advanced / Admin Use)

| Field | Required | Type | Description |
| --- | :---: | --- | --- |
| `ttl` | ✅ | number | Saga lifetime in seconds |
| `job` | ✅ | UUID | Saga job identifier — must be provided explicitly |
| `state` | ✅ | `SagaState` | `AWAITING`, `ABORTED`, `COMMITTED` |
| `session` | ✅ | hex string | Associated session identity |

#### Special Operations (Platform REST only — no MCP equivalent)

| Operation | HTTP | Purpose |
| --- | --- | --- |
| `start` | `POST /essential/sagas/start` | Initiate a saga (preferred) |
| `add` | `POST /essential/sagas/add` | Append a stage to an in-flight saga |
| `abort` | `GET /essential/sagas/:id/abort` | Abort an in-flight saga |
| `commit` | `GET /essential/sagas/:id/commit` | Commit a completed saga |

### `essential/saga-stages`

Records one step within a saga, including what was attempted and what came back.

> `essential/saga-stages` has **no MCP tools**. Stage data can only be read or written via platform REST.

#### Required Create Fields

| Field | Required | Type | Description |
| --- | :---: | --- | --- |
| `saga` | ✅ | MongoId | Parent saga ID |
| `bucket` | ✅ | `Collection` enum | Target collection for this stage |
| `action` | ✅ | `SagaStageAction` | `COUNT_DOCUMENTS`, `INSERT_MANY`, `FIND_ONE`, `FIND`, `FIND_ONE_AND_UPDATE`, `FIND_ONE_AND_DELETE`, `UPDATE_MANY` |
| `req` | ✅ | object | Captured request payload |

#### Population

`essential/saga-stages` supports `saga` → `EssentialSaga` plus common population.

### Key Behaviors

- Sagas are typically backend-driven. Client code starts them using `start`, then the Platform's `watcher` worker handles timeout compensation automatically.
- If a saga's TTL expires before it is committed, the `watcher` worker triggers compensation (rollback) of all recorded stages.
- The primary cross-service consumer is `financial/transactions` — every transaction is saga-linked.

## Identity

**Port:** REST `:3080` · gRPC `:5080`

Core user management service. Stores user accounts, extended profiles, and login sessions. All other services reference users by their `identity/users` MongoId.

### Collections

| Collection | Path | Purpose |
| --- | --- | --- |
| Users | `/identity/users` | Root user account record |
| Profiles | `/identity/profiles` | Extended personal / profile data |
| Sessions | `/identity/sessions` | Active login session records |

### `identity/users`

Root user account. Core access-control and ownership references across the Platform point to this collection.

#### Required Create Fields

| Field | Required | Type | Description |
| --- | :---: | --- | --- |
| `status` | ✅ | `Status` | Account lifecycle state: `ACTIVE`, `INACTIVE` |
| `subjects` | ✅ | `string[]` | At least one subject; values do **not** include domain suffix |
| `email` or `phone` | ✅ | `string` | At least one of these two must be present |

#### Optional Fields

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

#### Key Behaviors

- `password` and `secret` are write-only — they are never returned in responses.
- `subjects` stores values **without** the domain suffix (e.g. `guest`, not `guest@example.com`).
- Modifying `subjects` changes ABAC grant matching behavior.

### `identity/profiles`

Extended personal record linked through core ownership. A user typically has one profile.

#### Required Create Fields

| Field | Required | Type | Description |
| --- | :---: | --- | --- |
| `type` | ✅ | `ProfileType` | `REAL`, `LEGAL`, `GOVERN` |
| `gender` | ✅ | `Gender` | `MALE`, `FEMALE`, `UNKNOWN` |

#### Optional Fields

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

#### Population

Common population only. `cover`, `avatar`, and `gallery` are raw IDs — they are not populatable.

### `identity/sessions`

Login/session record with origin and expiration metadata.

#### Required Create Fields

| Field | Required | Type | Description |
| --- | :---: | --- | --- |
| `ip` | ✅ | string | Valid IPv4 or IPv6 address |
| `agent` | ✅ | string | User-agent string |
| `expiration` | ✅ | number | Unix timestamp |

#### Key Behaviors

- Sessions are soft-deleted on logout.
- The `cleaner` worker hard-deletes expired sessions.
- Use `?zone=own,client` to list a user's own sessions.

## Financial

**Port:** REST `:3060` · gRPC `:5060`

Manages the full financial lifecycle: accounts group wallets, wallets hold balances, invoices represent obligations, and transactions record atomic monetary movements. All transactions are saga-linked.

### Collections

| Collection | Path | Purpose |
| --- | --- | --- |
| Accounts | `/financial/accounts` | Groups one or more wallets |
| Currencies | `/financial/currencies` | Fiat, virtual, and chain currency definitions |
| Invoices | `/financial/invoices` | Billing documents with payees, line items, and optional recurrence |
| Transactions | `/financial/transactions` | Atomic monetary movement records |
| Wallets | `/financial/wallets` | Balance-holding wallet tied to one account and currency |

### `financial/accounts`

Groups one or more wallets and defines ownership semantics.

#### Required Create Fields

| Field | Required | Type | Description |
| --- | :---: | --- | --- |
| `type` | ✅ | `AccountType` | `CHECKING`, `SAVINGS`, `FIXED_DEPOSIT`, `MONEY_MARKET`, `JOINT`, `STUDENT`, `BUSINESS` |
| `ownership` | ✅ | `AccountOwnership` | `COMMON`, `PERSONAL`, `GOVERNMENT` |

#### Optional Fields

| Field | Type | Description |
| --- | --- | --- |
| `members` | MongoId[] | User IDs with access |

### `financial/currencies`

Fiat, virtual, and blockchain-related currency metadata.

#### Required Create Fields

| Field | Required | Type | Description |
| --- | :---: | --- | --- |
| `symbol` | ✅ | string | Display symbol |
| `type` | ✅ | `CurrencyType` | `REAL`, `VIRTUAL` |
| `precision` | ✅ | integer ≥ 0 | Decimal precision |

#### Optional Fields

| Field | Type | Description |
| --- | --- | --- |
| `code` | string | Short code |
| `name` | string | Display name |
| `countries` | string[] | ISO 3166-1 Alpha-3 codes |
| `token` | string | Token identifier |
| `network` | string | Blockchain network |
| `contract` | string | Contract address |
| `category` | `CurrencyCategory` | `COIN`, `TOKEN` |
| `lib` | `CurrencyLib` | Integration library |
| `nodes` | string[] | RPC node URLs |
| `subunits` | `CurrencyUnit[]` | Nested unit definitions |

### `financial/invoices`

Represents a financial obligation with payees, optional payers, line items, and optional recurrence.

#### Required Create Fields

| Field | Required | Type | Description |
| --- | :---: | --- | --- |
| `amount` | ✅ | number ≥ 0 | Total invoice amount |
| `payees` | ✅ | `Pay[]` | Non-empty recipient array |

#### Optional Fields

| Field | Type | Description |
| --- | --- | --- |
| `type` | `InvoiceType` | `TRANSACTION`, `REPEATABLE`, `REPLICATION`, `SUBSCRIPTION` |
| `title` | string | Invoice title |
| `payers` | `Pay[]` | Payment source wallets |
| `currency` | MongoId | `financial/currencies` |
| `items` | `InvoiceItem[]` | Line items |
| `discount` | number | Discount amount |
| `expires_at` | date | Expiration time |
| `subscription` | cron string | Recurrence rule (for `SUBSCRIPTION` type) |

#### Special Operation

| Operation | HTTP | Purpose |
| --- | --- | --- |
| `payment` | `GET /financial/invoices/:id/payment` | Convert invoice into a transaction flow |

> Invoice payment is a special operation, not a generic update. It requires the `payment:financial:invoices` scope.

#### Population

| Path | Collection |
| --- | --- |
| `currency` | `financial/currencies` |
| `payees.wallet` | `financial/wallets` |
| `payers.wallet` | `financial/wallets` |

### `financial/transactions`

Atomic monetary movement. Every transaction is linked to a saga.

> **`create` (raw CRUD) vs `init` (platform REST) are different operations.** Use `init` for normal payment flows.

#### `init` — Recommended for Payment Flows

```
POST /financial/transactions/init
```

| Field | Required | Type | Description |
| --- | :---: | --- | --- |
| `reason` | ✅ | `TransactionReason` | `SYNC`, `DEPOSIT`, `TRANSFER`, `WITHDRAW`, `PAYMENT` |
| `amount` | ✅ | number ≥ 0 | Amount |
| `saga` | — | MongoId | Attach to an existing saga |
| `payers` | — | `Pay[]` | Source wallets |
| `payees` | — | `Pay[]` | Destination wallets |
| `invoice` | — | MongoId | Linked invoice |

Platform manages `state`, `failed_at`, `verified_at`, `canceled_at` — never set these manually during `init`.

#### Special Operations

| Operation | HTTP | Purpose |
| --- | --- | --- |
| `abort` | `GET /financial/transactions/:id/abort` | Abort a pending/in-flight transaction |
| `verify` | `GET /financial/transactions/:id/verify` | Verify a transaction |

#### Population

| Path | Collection |
| --- | --- |
| `saga` | `essential/sagas` |
| `invoice` | `financial/invoices` |
| `payees.wallet` / `payers.wallet` | `financial/wallets` |

### `financial/wallets`

Digital wallet tied to one account and one currency.

#### Required Create Fields

| Field | Required | Type | Description |
| --- | :---: | --- | --- |
| `account` | ✅ | MongoId | `financial/accounts` |
| `currency` | ✅ | MongoId | `financial/currencies` |
| `amount` | ✅ | number | Primary balance |

#### Optional Fields

| Field | Type | Description |
| --- | --- | --- |
| `blocked` | number | Reserved / frozen balance |
| `internal` | number | Internal ledger balance |
| `external` | number | External ledger balance |
| `address` | string | Wallet address |
| `private` | string | Private key — write-only, never returned |

#### Population

| Path | Collection |
| --- | --- |
| `account` | `financial/accounts` |
| `currency` | `financial/currencies` |

### `Pay` — Shared Nested Object

Used in both invoices and transactions for wallet payment entries.

| Field | Required | Type | Description |
| --- | :---: | --- | --- |
| `type` | ✅ | `PayType` | `AMOUNT`, `BLOCKED`, `INTERNAL`, `EXTERNAL` |
| `wallet` | ✅ | MongoId | Target wallet |
| `amount` | — | number | Fixed amount |
| `fraction` | — | number | Proportional fraction (0–1) |

Use either `amount` or `fraction` for a given entry — not both.

## Career

**Port:** REST `:3140` · gRPC `:5140`

The largest domain service, managing all business-side entities: businesses, branches, employees, customers, products, services, stores, and inventory stock.

### Collections

| Collection | Path | Purpose |
| --- | --- | --- |
| Businesses | `/career/businesses` | Top-level business or organization |
| Branches | `/career/branches` | Business subdivisions |
| Employees | `/career/employees` | Staff records |
| Customers | `/career/customers` | Customer records |
| Products | `/career/products` | Product catalog items |
| Services | `/career/services` | Service offerings |
| Stores | `/career/stores` | Physical or logical stores / warehouses |
| Stocks | `/career/stocks` | Inventory records per product |

### `career/businesses`

Top-level business or organization record.

#### Required Create Fields

| Field | Required | Type | Description |
| --- | :---: | --- | --- |
| `name` | ✅ | string | Business name |
| `type` | ✅ | `BusinessType` | `SOLE_PROPRIETORSHIP`, `PARTNERSHIP`, `LLC`, `CORPORATION`, `COOPERATIVE`, `NONPROFIT` |
| `status` | ✅ | `Status` | `ACTIVE`, `INACTIVE` |

#### Common Optional Fields

| Field | Type | Description |
| --- | --- | --- |
| `code` | string | Short identifier |
| `alias` | string | Alternative name |
| `slogan` | string | Slogan |
| `website` | URL | Website URL |
| `address` | string | Address |
| `location` | MongoId | `logistic/locations` reference |
| `categories` | string[] | Category labels |
| `logo` / `cover` | MongoId | `special/files` references (raw, not populatable) |
| `foundation_date` | date | Foundation date |

#### Population

| Path | Collection |
| --- | --- |
| `location` | `logistic/locations` |

### `career/branches`

Business subdivision (main branch, warehouse, office, store, etc.).

#### Required Create Fields

| Field | Required | Type | Description |
| --- | :---: | --- | --- |
| `type` | ✅ | `BranchType` | `MAIN`, `SECONDARY`, `WAREHOUSE`, `OFFICE`, `STORE`, `VIRTUAL` |
| `business` | ✅ | MongoId | Parent business |
| `status` | ✅ | `Status` | `ACTIVE`, `INACTIVE` |

#### Population

| Path | Collection |
| --- | --- |
| `parent` | `career/branches` |
| `manager` | `career/employees` |
| `business` | `career/businesses` |
| `location` | `logistic/locations` |

### `career/employees`

Staff record linked to a business, optionally to a branch, manager, and services.

#### Required Create Fields

| Field | Required | Type | Description |
| --- | :---: | --- | --- |
| `type` | ✅ | `EmployeeType` | `PERMANENT`, `CONTRACT`, `PART_TIME`, `INTERN`, `FREELANCE` |
| `job_title` | ✅ | string | Role / job title |
| `business` | ✅ | MongoId | Parent business |
| `status` | ✅ | `Status` | `ACTIVE`, `INACTIVE` |

#### Population

| Path | Collection |
| --- | --- |
| `branch` | `career/branches` |
| `manager` | `career/employees` |
| `business` | `career/businesses` |
| `location` | `logistic/locations` |
| `currency` | `financial/currencies` |
| `profile` | `identity/profiles` |

### `career/customers`

Customer record associated with a business.

#### Required Create Fields

| Field | Required | Type | Description |
| --- | :---: | --- | --- |
| `type` | ✅ | `CustomerType` | `INDIVIDUAL`, `CORPORATE`, `GOVERNMENT`, `NONPROFIT` |
| `business` | ✅ | MongoId | Parent business |

#### Population

| Path | Collection |
| --- | --- |
| `profile` | `identity/profiles` |
| `stores` | `career/stores` |
| `branch` | `career/branches` |
| `services` | `career/services` |
| `business` | `career/businesses` |
| `employees` | `career/employees` |
| `location` / `addresses` | `logistic/locations` |
| `documents` / `certifications` | `special/files` |

### `career/products`

Catalog item offered by a business, branch, or store.

#### Required Create Fields

| Field | Required | Type | Description |
| --- | :---: | --- | --- |
| `name` | ✅ | string | Product name |

#### Nested `features` Object

Products may have a `features` array of nested objects — each with `type` (`COLOR`, `SIZE`, `WEIGHT`, `MATERIAL`, `STYLE`, `OTHER`), `title`, and `value`. Features are embedded objects, not referenced IDs.

#### Population

| Path | Collection |
| --- | --- |
| `store` | `career/stores` |
| `branch` | `career/branches` |
| `business` | `career/businesses` |
| `cover` / `gallery` | `special/files` |

### `career/services`

Service offering attached to a business.

#### Required Create Fields

| Field | Required | Type | Description |
| --- | :---: | --- | --- |
| `name` | ✅ | string | Service name |
| `type` | ✅ | `ServiceType` | `ONLINE`, `OFFLINE`, `HYBRID` |
| `status` | ✅ | `Status` | `ACTIVE`, `INACTIVE` |
| `business` | ✅ | MongoId | Parent business |

#### Population

| Path | Collection |
| --- | --- |
| `branch` | `career/branches` |
| `business` | `career/businesses` |
| `location` | `logistic/locations` |

### `career/stores`

Physical or logical store / warehouse.

#### Required Create Fields

| Field | Required | Type | Description |
| --- | :---: | --- | --- |
| `name` | ✅ | string | Store name |
| `type` | ✅ | `StoreType` | `RETAIL`, `WHOLESALE`, `WAREHOUSE`, `VIRTUAL` |
| `fork` | ✅ | `StoreFork` | Store fork classification |
| `business` | ✅ | MongoId | Parent business |

#### Population

| Path | Collection |
| --- | --- |
| `parent` | `career/stores` |
| `manager` | `career/employees` |
| `business` | `career/businesses` |
| `location` | `logistic/locations` |

### `career/stocks`

Inventory entry for a product.

#### Required Create Fields

| Field | Required | Type | Description |
| --- | :---: | --- | --- |
| `type` | ✅ | `StockType` | `AVAILABLE`, `RESERVED`, `DAMAGED`, `RETURNED`, `TRANSIT` |
| `product` | ✅ | MongoId | Parent product |
| `inventory` | ✅ | number | Current inventory level |

#### Population

| Path | Collection |
| --- | --- |
| `store` | `career/stores` |
| `branch` | `career/branches` |
| `product` | `career/products` |
| `business` | `career/businesses` |
| `location` | `logistic/locations` |
| `currency` | `financial/currencies` |

### Query Tips

- Scope most branch, employee, service, store, customer, product, and stock queries by `business`.
- Use `branch` for branch-local records.
- Use `store` for retail- or warehouse-local products and stocks.

## Special

**Port:** REST `:3090` · gRPC `:5090`

Handles file metadata and storage management (via MinIO) and time-dimensioned statistical counters.

### Collections

| Collection | Path | Purpose |
| --- | --- | --- |
| Files | `/special/files` | File metadata + MinIO object reference |
| Stats | `/special/stats` | Time-dimensioned counters and metrics |

### `special/files`

Stores file metadata — filename, MIME type, size, storage coordinates, and download URL. The binary content itself is stored in MinIO.

#### File Upload (Recommended Path)

Use the upload endpoints for new binary files — **not** the raw CRUD create:

| Endpoint | Scope | Description |
| --- | --- | --- |
| `POST /special/files/upload/private` | `upload:special:files` | Upload a private file |
| `POST /special/files/upload/public` | `upload:special:files` | Upload a public file |

Use raw `create` only when object-storage metadata already exists from a pre-signed or external upload flow.

#### Required Create Fields (raw create only)

| Field | Required | Type | Description |
| --- | :---: | --- | --- |
| `original` | ✅ | string | Original filename |
| `mimetype` | ✅ | string | MIME type |
| `size` | ✅ | number | File size in bytes |
| `bucket` | ✅ | string | Object-storage bucket |
| `key` | ✅ | string | Object-storage key/path |
| `acl` | ✅ | string | Storage access level |
| `location` | ✅ | string | File download URL |

#### Optional Fields

| Field | Type | Description |
| --- | --- | --- |
| `title` | string | Display title |
| `field` | string | Multipart field name |
| `content_type` | string | Download content-type override |
| `storage_class` | string | Storage class |
| `etag` | string | Storage ETag |
| `state` | `State` | Lifecycle state |

#### Special Operations (platform REST only)

| Operation | HTTP | Scope | Description |
| --- | --- | --- | --- |
| Share file | `POST /special/files/:id/share` | `share:special:files` | Generate a share link |
| Download file | `GET /special/files/download/:id` | `download:special:files` | Stream file contents |

### `special/stats`

Time-dimensioned counter and metric storage.

#### Required Create Fields

| Field | Required | Type | Description |
| --- | :---: | --- | --- |
| `type` | ✅ | `StatType` | `DAILY`, `MONTHLY`, `YEARLY` |
| `key` | ✅ | `StatKey` | Stat identifier |

#### Optional Fields

| Field | Type | Description |
| --- | --- | --- |
| `obj` | MongoId | Related document |
| `flag` | string | Secondary key |
| `day` | number | Day dimension |
| `month` | number | Month dimension |
| `year` | number | Year dimension |
| `hours` | number[] | Exactly 24 elements |
| `days` | number[] | Exactly 31 elements |
| `months` | number[] | Exactly 12 elements |

> `hours`, `days`, and `months` must always be provided with the exact required element count. Partial arrays are rejected.

#### Special Operations (platform REST only)

| Operation | HTTP | Scope | Description |
| --- | --- | --- | --- |
| Collect | `POST /special/stats/collect` | `collect:special:stats` | Upsert-style stat increment |
| Stackup | `POST /special/stats/stackup` | `collect:special:stats` | Aggregate / accumulate stats |

## Touch

**Port:** REST `:3100` · gRPC `:5100`

Records and dispatches all outbound communications — email, in-app notices, web push, and SMS. Every create or send operation may trigger real-world communication. Always confirm recipient, content, and channel before acting.

### Collections

| Collection | Path | Purpose |
| --- | --- | --- |
| Emails | `/touch/emails` | Outbound email records |
| Notices | `/touch/notices` | In-app notification records |
| Pushes | `/touch/pushes` | Web push subscription targets |
| Push Histories | `/touch/push-histories` | Push delivery attempt records |
| SMSs | `/touch/smss` | Outbound SMS records |

> `touch/push-histories` has **no MCP tools** — it is accessible via platform REST only.

### `touch/emails`

Stores outbound email envelope and body data.

#### Required Create Fields

| Field | Required | Type | Description |
| --- | :---: | --- | --- |
| `provider` | ✅ | `EmailProvider` | `NODEMAILER` |
| `to` | ✅ | string[] | Recipient email addresses |
| `from` | ✅ | string | Sender email (display name allowed) |
| `subject` | ✅ | string | Email subject |

#### Optional Fields

| Field | Type | Description |
| --- | --- | --- |
| `html` | string | HTML body |
| `text` | string | Plain text body |
| `cc` / `bcc` | string[] | Additional recipients |
| `reply_to` | string[] | Reply-to addresses |
| `date` | date | Scheduled send time |
| `attachments` | MongoId[] | Raw `special/files` IDs — not populatable |
| `smtp` | object | SMTP delivery metadata |

At least one of `html` or `text` must be present.

#### Special Send Operations (platform REST only)

```
POST /touch/emails/send
```

### `touch/notices`

In-app notifications displayed inside client applications.

#### Required Create Fields

| Field | Required | Type | Description |
| --- | :---: | --- | --- |
| `type` | ✅ | `NoticeType` | `INFO`, `EVENT`, `ALERT`, `WARNING`, `ERROR`, `SUCCESS`, `NOTICE`, `MESSAGE` |
| `title` | ✅ | string | Notice title |
| `content` | ✅ | string | Notice body |

#### Optional Fields

| Field | Type | Description |
| --- | --- | --- |
| `subtitle` | string | Secondary title |
| `category` | string | Custom grouping label |
| `visited` | boolean | Read status |
| `thumbnail` | MongoId | Raw file ID — not populatable |
| `attachments` | MongoId[] | Raw file IDs — not populatable |
| `actions` | object[] | Action buttons |

### `touch/pushes`

Web push subscription targets for browser/device delivery.

#### Required Create Fields

| Field | Required | Type | Description |
| --- | :---: | --- | --- |
| `session` | ✅ | MongoId | Session ID |
| `keys` | ✅ | object | Push subscription keys — **write-only, never returned** |
| `endpoint` | ✅ | string | Push endpoint URL |
| `expiration` | ✅ | number | Expiration Unix timestamp |

#### Special Send Operation (platform REST only)

```
POST /touch/pushes/send
```

### `touch/push-histories`

Delivery attempt records for push notifications.

> No MCP tools — use platform REST to inspect push delivery history.

#### Population

| Path | Collection |
| --- | --- |
| `to` | `touch/pushes` |

### `touch/smss`

Outbound SMS records.

#### Required Create Fields

| Field | Required | Type | Description |
| --- | :---: | --- | --- |
| `provider` | ✅ | `SmsProvider` | `KAVENEGAR`, `MELIPAYAMAK` |
| `message` | ✅ | string | SMS body |
| `receptors` | ✅ | string[] | Recipient phone numbers (E.164 format) |

#### Optional Fields

| Field | Type | Description |
| --- | --- | --- |
| `sender` | string | Sender number or line ID |
| `res` | object | Provider response metadata |

#### Special Send Operations (platform REST only)

| Endpoint | Purpose |
| --- | --- |
| `POST /touch/smss/send` | Direct SMS send |
| `POST /touch/smss/send/template` | Template SMS with parameters |

## Content

**Port:** REST `:3110` · gRPC `:5110`

Rich content management for user-facing documents: personal/shared notes, published posts and articles, and support tickets.

> Comments on posts and tickets live in `general/comments`, not in this service.

### Collections

| Collection | Path | Purpose |
| --- | --- | --- |
| Notes | `/content/notes` | Personal or shared notes and note threads |
| Posts | `/content/posts` | Published articles and content records |
| Tickets | `/content/tickets` | Support, issue, and task-like records |

### `content/notes`

Personal or shared notes. Notes can be threaded and can reference a post or ticket through `relation`.

#### Required Create Fields

| Field | Required | Type | Description |
| --- | :---: | --- | --- |
| `type` | ✅ | `NoteType` | `NOTE`, `REVIEW`, `COMMENT`, `FEEDBACK` |
| `content` | ✅ | string | Note body |

#### Optional Fields

| Field | Type | Description |
| --- | --- | --- |
| `parent` | MongoId | Parent note for threading |
| `level` | number | Thread nesting depth |
| `relation` | MongoId | Related `content/posts` or `content/tickets` ID |
| `visibility` | string | Visibility marker |
| `mentions` | MongoId[] | Raw user IDs — not populatable |
| `attachments` | MongoId[] | Raw file IDs — not populatable |
| `reactions` | object[] | Reaction objects |

#### Population

| Path | Collection |
| --- | --- |
| `relation` | `content/posts` or `content/tickets` |

### `content/posts`

Published content with a lifecycle centered on post status.

#### Required Create Fields

| Field | Required | Type | Description |
| --- | :---: | --- | --- |
| `title` | ✅ | string | Post title |
| `content` | ✅ | string | Post body |
| `status` | ✅ | `PostStatus` | `DRAFT`, `ARCHIVED`, `PUBLISHED` |

#### Optional Fields

| Field | Type | Description |
| --- | --- | --- |
| `slug` | string | URL-friendly identifier |
| `subtitle` | string | Subtitle |
| `summary` | string | Excerpt |
| `parent` | MongoId | Parent post |
| `categories` | string[] | Category values |
| `keywords` | string[] | SEO keywords |
| `thumbnail` | MongoId | Raw `special/files` ID — not populatable |
| `featured_image` | MongoId | Raw `special/files` ID — not populatable |
| `attachments` | MongoId[] | Raw file IDs — not populatable |
| `related_posts` | MongoId[] | Raw related post IDs — not populatable |
| `publication_date` | date | Scheduled or actual publication date |

#### Special Operation

`content/posts` exposes a service-local `search` operation in addition to standard CRUD:

```
POST /content/posts/search
```

Use `search` when full-text search behavior is needed rather than standard filtering.

### `content/tickets`

Support, issue, and task-like records with priority, lifecycle status, and optional resolution feedback.

#### Required Create Fields

| Field | Required | Type | Description |
| --- | :---: | --- | --- |
| `title` | ✅ | string | Ticket title |
| `content` | ✅ | string | Ticket body |
| `status` | ✅ | `TicketStatus` | `OPEN`, `CLOSED`, `RESOLVED`, `IN_PROGRESS` |
| `priority` | ✅ | `Priority` | `LOW`, `MEDIUM`, `HIGH`, `URGENT` |

#### Optional Fields

| Field | Type | Description |
| --- | --- | --- |
| `parent` | MongoId | Parent ticket |
| `department` | string | Routing department |
| `due_date` | date | Due date |
| `assigned_to` | MongoId | Assigned user ID (raw — not populatable) |
| `solution` | string | Resolution text |
| `feedback` | MongoId | `content/notes` ID used as formal resolution note |
| `attachments` | MongoId[] | Raw file IDs — not populatable |
| `related_tickets` | MongoId[] | Raw related ticket IDs — not populatable |

#### Population

| Path | Collection |
| --- | --- |
| `feedback` | `content/notes` |

### Query Tips

- Filter posts by `status` for publication lifecycle views.
- Filter tickets by `status` and `priority` for support queues.
- Use `relation` on notes to fetch all notes attached to a specific post or ticket.
- For comments on posts or tickets, query `general/comments` with the post or ticket ID.

## Logistic

**Port:** REST `:3120` · gRPC `:5120`

Tracks physical assets and movements — geographic locations, drivers, vehicles, cargoes, and travel routes. Integrates with OpenStreetMap for geocoding and the Valhalla engine for routing.

### Collections

| Collection | Path | Purpose |
| --- | --- | --- |
| Locations | `/logistic/locations` | GeoJSON places and areas |
| Drivers | `/logistic/drivers` | Driver records |
| Vehicles | `/logistic/vehicles` | Fleet vehicle records |
| Travels | `/logistic/travels` | Ordered trip records linking locations, drivers, vehicles, and cargoes |
| Cargoes | `/logistic/cargoes` | Shipment dimension and handling data |

### `logistic/locations`

GeoJSON-based physical places or areas (warehouses, depots, customer sites, checkpoints).

#### Required Create Fields

| Field | Required | Type | Description |
| --- | :---: | --- | --- |
| `geometry` | ✅ | object | GeoJSON geometry |

#### Geometry Fields

| Field | Required | Type | Description |
| --- | :---: | --- | --- |
| `type` | ✅ | `LocationGeometryType` | `Point`, `MultiPoint`, `LineString`, `MultiLineString`, `Polygon`, `MultiPolygon` |
| `coordinates` | ✅ | array | GeoJSON coordinates — **`[longitude, latitude]` order** |

> GeoJSON uses `[longitude, latitude]` — not `[latitude, longitude]`. This order is safety-critical.

#### Special Operations (platform REST only)

| Operation | HTTP | Scope | Description |
| --- | --- | --- | --- |
| Reverse geocode | `POST /logistic/locations/resolve/address` | `resolve:logistic:locations` | Coordinates → address |
| Forward geocode | `POST /logistic/locations/resolve/geocode` | `resolve:logistic:locations` | Text query → coordinates |

### `logistic/drivers`

Driver identity and operational status.

#### Required Create Fields

| Field | Required | Type | Description |
| --- | :---: | --- | --- |
| `type` | ✅ | `DriverType` | `CLASS_A`, `CLASS_B`, `CLASS_C`, `CLASS_M`, `CLASS_E` |
| `gender` | ✅ | `Gender` | `MALE`, `FEMALE`, `UNKNOWN` |
| `state` | ✅ | `State` | `PENDING`, `APPROVED`, `REJECTED`, `VERIFIED`, `UNKNOWN` |
| `status` | ✅ | `Status` | `ACTIVE`, `INACTIVE` |
| `license` | ✅ | string | License number |
| `expiration_date` | ✅ | date | License expiration date |

### `logistic/vehicles`

Fleet vehicle registry with assigned driver linkage.

#### Required Create Fields

| Field | Required | Type | Description |
| --- | :---: | --- | --- |
| `type` | ✅ | `VehicleType` | `CAR`, `TRUCK`, `MOTORCYCLE` |
| `status` | ✅ | `Status` | `ACTIVE`, `INACTIVE` |
| `plates` | ✅ | string[] | Plate numbers |

#### Population

| Path | Collection |
| --- | --- |
| `drivers` | `logistic/drivers` |

### `logistic/travels`

Ordered trip record linking route locations, cargoes, drivers, and vehicles.

#### Required Create Fields

| Field | Required | Type | Description |
| --- | :---: | --- | --- |
| `locations` | ✅ | MongoId[] | Ordered location IDs — first is origin, last is destination |

#### Optional Fields

| Field | Type | Description |
| --- | --- | --- |
| `cargoes` | MongoId[] | Assigned cargo IDs |
| `drivers` | MongoId[] | Assigned driver IDs |
| `vehicles` | MongoId[] | Assigned vehicle IDs |

#### Population

| Path | Collection |
| --- | --- |
| `locations` | `logistic/locations` |
| `cargoes` | `logistic/cargoes` |
| `drivers` | `logistic/drivers` |
| `vehicles` | `logistic/vehicles` |

#### Special Operation (platform REST only)

| Operation | HTTP | Scope | Description |
| --- | --- | --- | --- |
| Routing | `POST /logistic/travels/resolve/routing` | `resolve:logistic:travels` | Route planning via Valhalla |

Routing requires `service` (`route`, `isochrone`, `sources_to_targets`, `optimized_route`) and `options` (raw Valhalla payload).

### `logistic/cargoes`

Physical shipment metadata, dimensions, and handling flags.

#### Required Create Fields

| Field | Required | Type | Description |
| --- | :---: | --- | --- |
| `weight` | ✅ | number | Cargo weight |
| `width` | ✅ | number | Cargo width |
| `height` | ✅ | number | Cargo height |
| `length` | ✅ | number | Cargo length |

#### Optional Fields

| Field | Type | Description |
| --- | --- | --- |
| `title` | string | Human-readable label |
| `fragile` | boolean | Fragile handling flag |
| `perishable` | boolean | Perishable handling flag |
| `travels` | MongoId[] | Linked travel IDs |

#### Population

| Path | Collection |
| --- | --- |
| `travels` | `logistic/travels` |

## Conjoint

**Port:** REST `:3130` · gRPC `:5130`

Real-time messaging infrastructure. Manages messaging identities (accounts), conversation spaces (channels), contact directories, channel memberships, and messages. Message delivery uses EMQX/MQTT via the `publisher` worker.

### Collections

| Collection | Path | Purpose |
| --- | --- | --- |
| Accounts | `/conjoint/accounts` | Messaging identity |
| Channels | `/conjoint/channels` | Conversation container |
| Contacts | `/conjoint/contacts` | Contact directory entries |
| Members | `/conjoint/members` | Account-to-channel membership |
| Messages | `/conjoint/messages` | Messages inside a channel |

### `conjoint/accounts`

A messaging identity that can participate in channels and messages.

#### Required Create Fields

| Field | Required | Type | Description |
| --- | :---: | --- | --- |
| `type` | ✅ | `AccountType` | `HUMAN`, `ROBOT` |

#### Optional Fields

| Field | Type | Description |
| --- | --- | --- |
| `profile` | MongoId | `identity/profiles` reference |
| `bio` | string | Short bio text |
| `status` | string | Local status string |

#### Population

| Path | Collection |
| --- | --- |
| `profile` | `identity/profiles` |

### `conjoint/channels`

A conversation space — group, broadcast stream, or direct conversation.

#### Required Create Fields

| Field | Required | Type | Description |
| --- | :---: | --- | --- |
| `type` | ✅ | `ChannelType` | `GROUP`, `BROADCAST`, `CONVERSATION` |
| `scope` | ✅ | `ChannelScope` | `PUBLIC`, `PRIVATE`, `PROTECTED` |

#### Optional Fields

| Field | Type | Description |
| --- | --- | --- |
| `name` | string | Short channel name |
| `title` | string | Human-readable title |
| `profile` | MongoId | `identity/profiles` reference |
| `account` | MongoId | Owning `conjoint/accounts` |
| `pinned_messages` | MongoId[] | Pinned message IDs (raw, not populatable) |

#### Population

| Path | Collection |
| --- | --- |
| `profile` | `identity/profiles` |
| `account` | `conjoint/accounts` |

### `conjoint/contacts`

Contact directory entry, optionally linked to a messaging account.

#### Required Create Fields

| Field | Required | Type | Description |
| --- | :---: | --- | --- |
| `type` | ✅ | `ContactType` | `MAIN`, `HOME`, `WORK`, `OTHER` |

#### Optional Fields

| Field | Type | Description |
| --- | --- | --- |
| `phone` | string | Phone number |
| `email` | string | Email address |
| `account` | MongoId | `conjoint/accounts` reference |
| `nickname` | string | Display nickname |

#### Population

| Path | Collection |
| --- | --- |
| `account` | `conjoint/accounts` |

### `conjoint/members`

Joins an account to a channel with optional role and permissions.

#### Required Create Fields

| Field | Required | Type | Description |
| --- | :---: | --- | --- |
| `account` | ✅ | MongoId | `conjoint/accounts` |
| `channel` | ✅ | MongoId | `conjoint/channels` |

#### Optional Fields

| Field | Type | Description |
| --- | --- | --- |
| `role` | string | Membership role label |
| `permissions` | string[] | Fine-grained permission labels |

> The `{ account, channel }` pair has a unique compound index — duplicate memberships are not valid.

#### Population

| Path | Collection |
| --- | --- |
| `account` | `conjoint/accounts` |
| `channel` | `conjoint/channels` |

### `conjoint/messages`

Polymorphic messages inside a channel.

#### Required Create Fields

| Field | Required | Type | Description |
| --- | :---: | --- | --- |
| `type` | ✅ | `MessageType` | `TEXT`, `FILE`, `IMAGE`, `VIDEO`, `AUDIO`, `STICKER`, `GALLERY`, `CONTACT`, `COMMAND`, `DOCUMENT`, `LOCATION`, `PULL` |
| `content` | ✅ | any | Polymorphic message body |

#### Optional Fields

| Field | Type | Description |
| --- | --- | --- |
| `account` | MongoId | Sender account |
| `channel` | MongoId | Target channel |
| `caption` | string | Optional caption |
| `mentions` | MongoId[] | Mentioned **account** IDs (not user IDs) |
| `reply_to` | MongoId | Parent message ID (raw, not populatable) |
| `hashtags` | string[] | Hashtag strings |
| `reactions` | object[] | Reaction objects |
| `scheduled_at` | date | Scheduled send timestamp |
| `forwarded_from` | MongoId | Forwarding account ID |
| `originate_from` | MongoId | Original sender account ID |

> `mentions` are `conjoint/accounts` IDs — not `identity/users` IDs.

#### Population

| Path | Collection |
| --- | --- |
| `account` | `conjoint/accounts` |
| `channel` | `conjoint/channels` |
| `mentions` | `conjoint/accounts` |
| `originate_from` | `conjoint/accounts` |
| `forwarded_from` | `conjoint/accounts` |

### Query Tips

- Scope message queries by `channel` for conversation history.
- Use `created_at` descending sort for message history pagination.
- Query memberships by `channel` to list members, or by `account` to list channel memberships.

## General

**Port:** REST `:3070` · gRPC `:5070`

Shared cross-cutting entities used across multiple domains: audit activities, typed key-value artifacts, threaded comments, calendar events, and BPMN-style workflow state.

### Collections

| Collection | Path | Purpose |
| --- | --- | --- |
| Activities | `/general/activities` | Audit and activity log entries |
| Artifacts | `/general/artifacts` | Typed key-value or JSON-like stored artifacts |
| Comments | `/general/comments` | Threaded comments for posts and tickets |
| Events | `/general/events` | Calendar-like events, meetings, and deadlines |
| Workflows | `/general/workflows` | BPMN-style process state and token history |

### `general/activities`

Append-heavy audit and activity records. Treat as write-once — prefer creating new activities over mutating old ones.

#### Required Create Fields

| Field | Required | Type | Description |
| --- | :---: | --- | --- |
| `type` | ✅ | `ActivityType` | `USER`, `SYSTEM`, `EXTERNAL` |
| `message` | ✅ | string | Human-readable summary |

#### Optional Fields

| Field | Type | Description |
| --- | --- | --- |
| `source` | string | Originating service or module |
| `details` | object | Structured payload |
| `metadata` | object | Extra structured context |

### `general/artifacts`

Typed key-value storage for generated reports, serialized outputs, exports, and flexible JSON payloads.

#### Required Create Fields

| Field | Required | Type | Description |
| --- | :---: | --- | --- |
| `key` | ✅ | string | Artifact key — uppercase, underscore, digit, dash, and dot pattern |
| `type` | ✅ | `ValueType` | `NULL`, `ARRAY`, `OBJECT`, `STRING`, `NUMBER`, `BOOLEAN` |

#### Optional Fields

| Field | Type | Description |
| --- | --- | --- |
| `value` | object | Artifact payload |

### `general/comments`

Threaded discussion attached to `content/posts` or `content/tickets`. Comments live in this service, not in the `content` service.

#### Required Create Fields

| Field | Required | Type | Description |
| --- | :---: | --- | --- |
| `content` | ✅ | string | Comment body |

#### Optional Fields

| Field | Type | Description |
| --- | --- | --- |
| `post` | MongoId | Target `content/posts` document |
| `ticket` | MongoId | Target `content/tickets` document |
| `parent` | MongoId | Parent comment for threading |
| `level` | integer | Nesting depth |
| `status` | `CommentStatus` | `DRAFT`, `ARCHIVED`, `PUBLISHED` |
| `mentions` | MongoId[] | Raw mentioned IDs — not populatable |
| `attachments` | MongoId[] | Raw file IDs — not populatable |
| `reactions` | object[] | Reaction objects |

#### Population

| Path | Collection |
| --- | --- |
| `post` | `content/posts` |
| `ticket` | `content/tickets` |
| `parent` | `general/comments` |

### `general/events`

Scheduled meetings, deadlines, reminders, and other bounded events.

#### Required Create Fields

| Field | Required | Type | Description |
| --- | :---: | --- | --- |
| `title` | ✅ | string | Event title |
| `s_date` | ✅ | date | Start datetime |
| `e_date` | ✅ | date | End datetime — must be after `s_date` |

#### Optional Fields

| Field | Type | Description |
| --- | --- | --- |
| `subtitle` | string | Subtitle |
| `place` | string | Venue label |
| `location` | MongoId | Raw `logistic/locations` ID — not populatable |
| `attendees` | MongoId[] | `identity/profiles` IDs |
| `organizers` | MongoId[] | `career/employees` IDs |
| `category` | string | Category label |
| `correlation` | UUID | Groups recurring event instances |

#### Population

| Path | Collection |
| --- | --- |
| `attendees` | `identity/profiles` |
| `organizers` | `career/employees` |

### `general/workflows`

BPMN-style workflow execution state, including overall status and per-token history.

#### Required Create Fields

| Field | Required | Type | Description |
| --- | :---: | --- | --- |
| `name` | ✅ | string | Workflow name |
| `status` | ✅ | `WorkflowStatus` | `ready`, `paused`, `failed`, `running`, `completed`, `terminated` |
| `tokens` | ✅ | `WorkflowToken[]` | Execution threads |

#### `WorkflowToken` Structure

| Field | Required | Type | Description |
| --- | :---: | --- | --- |
| `id` | ✅ | string | Token identifier |
| `history` | ✅ | `WorkflowState[]` | Ordered state history |
| `parent` | — | string | Parent token ID |
| `locked` | — | boolean | Rollback guard |

Each `WorkflowState` entry has `ref` (BPMN element reference), `status`, optional `name`, and optional `value`.

> Preserve token history carefully — do not guess workflow state transitions or invent BPMN element references.

## Thing

**Port:** REST `:3150` · gRPC `:5150`

Internet of Things device management and sensor telemetry. Manages physical devices, their measurement channels (sensors), and the time-series readings (metrics) those sensors produce.

### Collections

| Collection | Path | Purpose |
| --- | --- | --- |
| Devices | `/thing/devices` | Physical IoT device or gateway records |
| Sensors | `/thing/sensors` | Measurement channels attached to devices |
| Metrics | `/thing/metrics` | Time-series telemetry readings |

### `thing/devices`

A physical IoT unit or gateway.

#### Required Create Fields

| Field | Required | Type | Description |
| --- | :---: | --- | --- |
| `name` | ✅ | string | Human-readable device name |

#### Optional Fields

| Field | Type | Description |
| --- | --- | --- |
| `type` | string | Device type / category |
| `token` | string | Device communication token |
| `state` | `State` | `PENDING`, `APPROVED`, `REJECTED`, `VERIFIED`, `UNKNOWN` |
| `status` | `Status` | `ACTIVE`, `INACTIVE` |
| `location` | MongoId | Raw `logistic/locations` ID — **not populatable** |

> `thing/devices.location` is a raw MongoId. To resolve location details, read the ID and query `logistic/locations` separately.

### `thing/sensors`

A logical measurement channel attached to a device.

#### Required Create Fields

| Field | Required | Type | Description |
| --- | :---: | --- | --- |
| `device` | ✅ | MongoId | Parent device ID |

#### Optional Fields

| Field | Type | Description |
| --- | --- | --- |
| `name` | string | Sensor name |
| `type` | string | Sensor category |
| `unit` | string | Measurement unit (e.g. `°C`, `hPa`) |
| `metric` | string | Metric label / key family |
| `state` | `State` | Operational state |
| `status` | `Status` | Administrative status |

#### Population

| Path | Collection |
| --- | --- |
| `device` | `thing/devices` |

### `thing/metrics`

A telemetry reading produced by a sensor.

#### Required Create Fields

| Field | Required | Type | Description |
| --- | :---: | --- | --- |
| `sensor` | ✅ | MongoId | Source sensor ID |
| `value` | ✅ | `number \| number[]` | Scalar or numeric array reading |

#### Optional Fields

| Field | Type | Description |
| --- | --- | --- |
| `key` | string | Metric key identifier |
| `state` | `State` | Reading state classification |
| `device` | MongoId | Direct device reference for efficient filtering — **write-only in responses** |

> `thing/metrics.device` is write-only in serializer responses. It will not appear in returned payloads even if provided on create. To get device context, populate `sensor` and resolve through it.

#### Population

| Path | Collection |
| --- | --- |
| `sensor` | `thing/sensors` |
| `device` | `thing/devices` |

### Query Tips

- Always filter metrics by `sensor`, `device`, `key`, and/or a `created_at` time range — metric collections grow quickly.
- Use `count` before paginating high-frequency metric datasets.
- Sort recent metrics by `created_at: desc`.

## Scope Naming Convention

Required scopes follow the pattern `{action}:{service}:{collection}`:

```
read:identity:users       → ReadIdentityUsers
write:financial:accounts  → WriteFinancialAccounts
manage:auth:apts          → ManageAuthApts
```

The `manage:` prefix grants all read, write, and destructive actions including `destroy` and bulk operations.

## Health Checks

Every service exposes `GET /status` with checks for its dependencies:

```bash
curl http://localhost:3020/status    # auth
curl http://localhost:3080/status    # identity
curl http://localhost:3060/status    # financial
```

Example response:

```json
{
  "status": "ok",
  "info": {
    "mongodb": { "status": "up" },
    "redis": { "status": "up" },
    "kafka": { "status": "up" }
  }
}
```
