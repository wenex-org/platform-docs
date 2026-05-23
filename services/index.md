# Service Catalog

Wenex Platform consists of 15 domain microservices plus 7 worker processes. All services are NestJS applications that expose a REST API and a gRPC server. Workers are event-driven consumers that do not expose a public REST API.

---

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

    subgraph Workers
        DISP["dispatcher<br/>:4010"]
        OBS["observer<br/>:4020"]
        PRES["preserver<br/>:4030"]
        WATCH["watcher<br/>:4040"]
        PUB["publisher<br/>:4050"]
        LOGG["logger<br/>:4060"]
        CLN["cleaner<br/>:4070"]
    end

    GW[Gateway :3010] --> Core Services
    GW --> Business Services
    Core Services --> Kafka[(Kafka)]
    Business Services --> Kafka
    Kafka --> Workers
```

---

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

---

## Core Services

### auth — Authentication & Authorization

**Port:** REST `:3020` · gRPC `:5020`

Handles all authentication flows and token management. Unique in that it does not follow the standard CRUD pattern for its primary `auth` endpoint.

| Collection | Path | Purpose |
|---|---|---|
| Auths | `/auth` | `token`, `verify`, `logout`, `check`, `can` |
| APTs | `/auth/apts` | Long-lived Auth Personal Tokens (API keys) |
| Grants | `/auth/grants` | OAuth permission grants |

**Key behaviors:**
- `POST /auth/token` is public (`@IsPublic()`) — no auth header required
- APTs are revocable and scoped — create them with minimal permissions
- `POST /auth/can` performs ABAC policy evaluation via the `abacl` library

---

### domain — Tenant & OAuth Management

**Port:** REST `:3030` · gRPC `:5030`

Manages tenant domains, OAuth applications, and client registrations.

| Collection | Path | Purpose |
|---|---|---|
| Apps | `/domain/apps` | OAuth application definitions |
| Clients | `/domain/clients` | OAuth client credentials |

---

### context — Configuration & Settings

**Port:** REST `:3040` · gRPC `:5040`

Stores application-wide and per-user configuration data.

| Collection | Path | Purpose |
|---|---|---|
| Configs | `/context/configs` | Application configuration entries |
| Settings | `/context/settings` | User/tenant preference settings |

**Key behaviors:**

- `context/configs` is the registry for all CQRS webhook endpoints. Each client registers its own entry using `key: ConfigKey.CQRS` and `eid: <client_id>`:

```typescript
await platform.context.configs.create({
  key: 'CQRS',                              // ConfigKey.CQRS
  eid: '68fc7a456e8fa60ae29c3d02',          // this client's MongoDB _id (= client_id in tokens)
  value: { webhook: 'http://localhost:8150/cqrs' },
});
```

The Platform's `publisher` worker reads these entries after every write to deliver webhook payloads to each client listed in a document's `clients[]` array.

---

### essential — Saga Orchestration

**Port:** REST `:3050` · gRPC `:5050`

Manages distributed saga transactions across multiple services. Workers use PostgreSQL-backed saga stages for durability.

| Collection | Path | Purpose |
|---|---|---|
| Sagas | `/essential/sagas` | Distributed saga instances |
| Saga Stages | `/essential/saga-stages` | Individual compensating steps |

**Key behaviors:**
- Sagas coordinate multi-service operations (e.g., order creation touching financial + logistic + conjoint)
- PostgreSQL stores saga stage state for crash recovery
- Workers (`dispatcher`, `observer`, `preserver`) drive saga execution

---

### identity — Users, Profiles & Sessions

**Port:** REST `:3080` · gRPC `:5080`

Core user management service. All other services reference users by MongoId.

| Collection | Path | Purpose |
|---|---|---|
| Users | `/identity/users` | User accounts |
| Profiles | `/identity/profiles` | Extended profile data |
| Sessions | `/identity/sessions` | Active login sessions |

**Key behaviors:**
- User creation triggers Kafka events consumed by other services (e.g., wallet auto-creation in `financial`)
- Sessions are soft-deleted on logout, hard-deleted by the `cleaner` worker

---

## Business Services

### financial — Billing & Payments

**Port:** REST `:3060` · gRPC `:5060`

Manages the full financial lifecycle: accounts → wallets → invoices → transactions.

| Collection | Path | Purpose |
|---|---|---|
| Accounts | `/financial/accounts` | Financial account records |
| Wallets | `/financial/wallets` | Balance-holding wallets |
| Invoices | `/financial/invoices` | Billing documents |
| Transactions | `/financial/transactions` | Payment / transfer records |
| Currencies | `/financial/currencies` | Supported currency definitions |

---

### career — Business Operations

**Port:** REST `:3140` · gRPC `:5140`

The largest domain service, managing all business-side entities.

| Collection | Path | Purpose |
|---|---|---|
| Businesses | `/career/businesses` | Business entity (top-level) |
| Branches | `/career/branches` | Physical/logical branches |
| Employees | `/career/employees` | Staff records |
| Products | `/career/products` | Product catalog |
| Services | `/career/services` | Offered services |
| Stocks | `/career/stocks` | Inventory per product/branch |
| Stores | `/career/stores` | Retail/online store fronts |
| Customers | `/career/customers` | Customer records |

---

### special — Files & Statistics

**Port:** REST `:3090` · gRPC `:5090`

Handles file uploads (via MinIO) and computed statistics.

| Collection | Path | Purpose |
|---|---|---|
| Files | `/special/files` | File metadata + MinIO reference |
| Stats | `/special/stats` | Aggregated metrics / counters |

**Key behaviors:**
- File content is stored in MinIO; only metadata (filename, size, mime, url) is in MongoDB
- The `publisher` worker processes file events for CDN propagation

---

### touch — Notifications & Messaging

**Port:** REST `:3100` · gRPC `:5100`

Records and dispatches all outbound communications.

| Collection | Path | Purpose |
|---|---|---|
| Emails | `/touch/emails` | Outbound email records |
| Notices | `/touch/notices` | In-app notifications |
| Pushes | `/touch/pushes` | Mobile push records |
| SMS | `/touch/sms` | SMS records |
| Histories | `/touch/histories` | Delivery status history |

**Key behaviors:**
- Records are created first; actual delivery is handled by the `publisher` worker via EMQX/MQTT
- `histories` tracks per-recipient delivery status

---

### content — Notes, Posts & Support

**Port:** REST `:3110` · gRPC `:5110`

Rich content management for user-facing documents.

| Collection | Path | Purpose |
|---|---|---|
| Notes | `/content/notes` | Rich-text notes / documents |
| Posts | `/content/posts` | Published articles / blog posts |
| Tickets | `/content/tickets` | Support ticket threads |

---

### logistic — Tracking & Delivery

**Port:** REST `:3120` · gRPC `:5120`

Tracks physical assets and movements.

| Collection | Path | Purpose |
|---|---|---|
| Locations | `/logistic/locations` | Geographic coordinates / addresses |
| Drivers | `/logistic/drivers` | Driver profiles |
| Vehicles | `/logistic/vehicles` | Vehicle registry |
| Travels | `/logistic/travels` | Trip / route records |
| Cargoes | `/logistic/cargoes` | Shipment manifests |

---

### conjoint — Messaging & Channels

**Port:** REST `:3130` · gRPC `:5130`

Real-time messaging infrastructure.

| Collection | Path | Purpose |
|---|---|---|
| Accounts | `/conjoint/accounts` | Messaging account (bot/user) |
| Channels | `/conjoint/channels` | Conversation channels |
| Contacts | `/conjoint/contacts` | Contact directory |
| Members | `/conjoint/members` | Channel membership records |
| Messages | `/conjoint/messages` | Individual messages |

**Key behaviors:**
- Message delivery uses EMQX/MQTT via the `publisher` worker
- Channels support multiple account types (user, bot, business)

---

### general — Cross-Cutting Entities

**Port:** REST `:3070` · gRPC `:5070`

Shared entities used across domains.

| Collection | Path | Purpose |
|---|---|---|
| Activities | `/general/activities` | Audit activity log entries |
| Artifacts | `/general/artifacts` | Generic binary references |
| Comments | `/general/comments` | Threaded comment system |
| Events | `/general/events` | Calendar / system events |
| Workflows | `/general/workflows` | Process definitions |

---

### thing — IoT & Telemetry

**Port:** REST `:3150` · gRPC `:5150`

Internet of Things device management and sensor telemetry.

| Collection | Path | Purpose |
|---|---|---|
| Devices | `/thing/devices` | IoT device registry |
| Sensors | `/thing/sensors` | Sensor definitions per device |
| Metrics | `/thing/metrics` | Sensor readings / time-series |

---

## Worker Processes

Workers are internal consumers — they have no public REST API. They expose only `/status` (health) and `/metrics` (Prometheus).

| Worker | Port | Role |
|---|---|---|
| **dispatcher** | 4010 | Main job queue: receives Kafka events, dispatches BullMQ jobs to other workers |
| **observer** | 4020 | Observes domain events: triggers notifications, audit logs |
| **preserver** | 4030 | Persistence worker: syncs Elasticsearch, snapshot creation |
| **watcher** | 4040 | Monitors saga execution, triggers compensations on timeout |
| **publisher** | 4050 | Outbound delivery: EMQX/MQTT push, email, SMS relay |
| **logger** | 4060 | Aggregates logs from all services, ships to Elasticsearch |
| **cleaner** | 4070 | Data retention: hard-deletes expired soft-deleted records |

### Worker Internal Structure

```
apps/workers/<name>/src/
├── main.ts              # Kafka consumer bootstrap + HTTP health server
├── app.module.ts        # Kafka, BullMQ, Mongo, Postgres imports
├── app.processor.ts     # @Process() — BullMQ job handler
├── app.task.ts          # @Cron() — scheduled maintenance tasks
├── app.service.ts       # Core business logic
├── app.controller.ts    # HTTP: /status, /metrics
└── entities/            # TypeORM entities (PostgreSQL)
```

---

## Scope Naming Convention

Required scopes follow the pattern `{action}:{service}:{collection}`:

```
read:identity:users       → ReadIdentityUsers
write:financial:accounts  → WriteFinancialAccounts
manage:auth:apts          → ManageAuthApts
```

The `manage:` prefix grants all read, write, and destructive actions including `destroy` and bulk operations.

---

## Health Checks

Every service and worker exposes `GET /status` with checks for its dependencies:

```bash
curl http://localhost:3020/status    # auth
curl http://localhost:3080/status    # identity
curl http://localhost:4010/status    # dispatcher worker
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
