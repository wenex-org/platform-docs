# Setup

Quick-start guide for running the Wenex platform.

## Prerequisites

**Git** is required to clone the repository and manage submodules. Any recent version works.

**Docker and Docker Compose** are required to run the infrastructure stack. Any recent version works.

**Node.js 22.x and pnpm 10.5.2** are required for the platform itself. The repository includes an `.nvmrc` file pinning the exact Node version. Use [nvm](https://github.com/nvm-sh/nvm) to install and activate it, then enable [corepack](https://nodejs.org/api/corepack.html) so the correct pnpm version is available automatically:

```bash
nvm install   # installs the Node version from .nvmrc
nvm use       # switches to it
corepack enable
```

## Start Infrastructure

```bash
docker-compose -f docker/docker-compose.yml up -d
```

This starts the full core stack: MongoDB (replica set), PostgreSQL, Redis, Kafka, Zookeeper, Elasticsearch, MinIO, and EMQX.

### Additional Compose

The `docker/` directory includes separate files for optional or standalone services:

| File | Service | Notes |
| --- | --- | --- |
| `docker-compose.otlp.yml` | OpenTelemetry | Distributed tracing and metrics collection. |
| `docker-compose.osm.yml` | Nominatim + Tileserver | Self-hosted geocoding (Nominatim) and map tile serving (Tileserver). Use `--profile` to control which components start. |
| `docker-compose.snq.yml` | SonarQube | Static code analysis and quality gate. |
| `docker-compose.sntr.yml` | Sentry | Error monitoring and tracking. |
| `docker-compose.turn.yml` | TURN Server | WebRTC relay for NAT traversal. |

For the initial tile data import, run the `setup` profile first:

```bash
docker-compose -f docker/docker-compose.osm.yml --profile setup up -d
```

Then start the tileserver with the `start` profile:

```bash
docker-compose -f docker/docker-compose.osm.yml --profile start up -d
```

### MongoDB Replica Set

The main compose file runs MongoDB as a replica set. Add these entries to `/etc/hosts` so the hostnames resolve correctly:

```text
127.0.0.1 mongodb-primary
127.0.0.1 mongodb-secondary
```

Connection URL:

```text
mongodb://root:password123@mongodb-primary:27017,mongodb-secondary:27018/?replicaSet=rs0&authSource=admin
```

## Manually Setup

### Clone and Install

Clone the repository:

```sh
git clone git@github.com:wenex-org/platform.git
cd platform
```

Copy the environment template and pull the submodules:

```sh
cp .env.example .env
npm run git:clone
npm run git checkout main
```

Install Node dependencies from lock file:

```sh
pnpm install --frozen-lockfile
```

### Initialization

Run these commands once before starting any platform service. Each one must complete successfully before the platform can operate correctly.

Seed MongoDB with the root user, app, client, grant, CQRS config, currencies, and wallets:

```sh
npm run db:seed
```

Create Elasticsearch indices for messages, posts, and products, and initialize the PostgreSQL database:

```sh
npm run db:index
```

Create the MinIO `public` and `private` buckets:

```sh
npm run storage:init
```

Register the EMQX ExHook server and remove the default file-based authorization source:

```sh
npm run utility:init
```

::: tip
To wipe all data and start fresh, use `npm run db:clean`. This permanently drops all MongoDB collections, Elasticsearch indices, and Redis keys — use with caution.
:::

### Kafka Connect

This step is required before starting platform services. It registers a [Debezium](https://debezium.io) MongoDB source connector with Kafka Connect, enabling Change Data Capture (CDC) across all platform collections.

The connector monitors every write to the MongoDB databases (auth, identity, domain, context, financial, career, conjoint, content, logistic, general, special, touch, thing, and essential sagas) and publishes change events to Kafka under the `mongo.` topic prefix. Workers such as `dispatcher`, `observer`, and `publisher` consume these events to drive notifications, audit logs, webhooks, and saga orchestration.

Run after the infrastructure and Kafka Connect container are up:

```sh
npm run script:kafka-connect
```

If the connector already exists it is automatically removed and re-registered, so this command is safe to re-run.

### Start Platform

Start each group in a separate terminal. Services must be fully up before starting the gateway; workers can start in parallel with the gateway.

```sh
npm run start:dev <project-name>
```

**Services** — internal apps that expose gRPC servers consumed by the gateway via Protobuf:

| Project | Description |
| --- | --- |
| `auth` | Authentication, token issuance, APTs, OAuth grants, and ABAC policy evaluation |
| `domain` | Tenant management, OAuth applications and client credentials |
| `context` | Application configs and user settings; hosts the CQRS webhook registry |
| `essential` | Distributed saga orchestration with PostgreSQL-backed compensating steps |
| `identity` | Users, profiles, and login sessions |
| `financial` | Accounts, wallets, invoices, transactions, and currencies |
| `career` | Businesses, branches, employees, products, services, stocks, stores, and customers |
| `special` | File uploads (MinIO) and aggregated statistics |
| `touch` | Outbound communications: email, SMS, push notifications, and in-app notices |
| `content` | Notes, published posts, and support tickets |
| `logistic` | Locations, drivers, vehicles, trips, and cargo shipments |
| `conjoint` | Real-time messaging: accounts, channels, contacts, members, and messages |
| `general` | Cross-cutting entities: activities, artifacts, comments, events, and workflows |
| `thing` | IoT device registry, sensor definitions, and time-series metrics |

**Gateway** — the public entry point that exposes REST, GraphQL, and gRPC and routes traffic to the services above:

| Project | Description |
| --- | --- |
| `gateway` | Starts the unified gateway (default port: `3010`) |

**Workers** — background Kafka consumers driven by CDC events; no public REST API:

| Project | Description |
| --- | --- |
| `dispatcher` | Receives Kafka events and dispatches BullMQ jobs to other workers |
| `observer` | Triggers notifications and audit log entries on domain events |
| `preserver` | Syncs documents to Elasticsearch and handles snapshot creation |
| `watcher` | Monitors saga timeouts and triggers compensating transactions |
| `publisher` | Delivers outbound messages via EMQX/MQTT, email, and SMS |
| `logger` | Aggregates logs from all services and ships them to Elasticsearch |
| `cleaner` | Hard-deletes expired soft-deleted records on a scheduled basis |

## Docker Setup

Run the full platform (gateway, services, and workers) as Docker containers using the root `docker-compose.yml`. Each container uses the same image — the `SERVICE_NAME` environment variable selects which process to start.

### Prepare the Environment

Copy the environment template and generate a unique machine ID:

```bash
cp .env.example .env
npm run script:machine
```

`script:machine` writes a random `MACHINE_ID` into `.env` — required before starting any container.

### Build the Platform Image

From the repository root:

```bash
docker build -t wenex/platform:latest .
```

This runs `npm run git:clone`, `pnpm install --frozen-lockfile`, and `npm run script:build` inside the container, producing a self-contained image with all compiled services and workers under `wnx/`.

### Initialize the Database

Run the initialization commands once using the compose profiles before starting the platform for the first time:

```bash
docker-compose run --rm db-seed
docker-compose run --rm db-index
docker-compose run --rm storage-init
docker-compose run --rm utility-init
docker-compose run --rm kafka-connect
```

::: tip
To wipe all data and start fresh, use `docker-compose run --rm db-clean`. This permanently drops all MongoDB collections, Elasticsearch indices, and Redis keys — use with caution.
:::

### Start the Platform

The `docker-compose.yml` uses Docker Compose profiles to control which containers start. Use the `platform` profile to bring up the full stack — all services, workers, and the gateway:

```bash
docker-compose --profile platform up -d
```

To start only the services (no workers):

```bash
docker-compose --profile services up -d
```

To start only the workers:

```bash
docker-compose --profile workers up -d
```

To start a single component by name:

```bash
docker-compose --profile gateway up -d
docker-compose --profile auth up -d
docker-compose --profile dispatcher up -d
```

::: tip
Make sure the infrastructure stack is running first (`docker-compose -f docker/docker-compose.yml up -d`) before starting the platform containers. Services and workers expect MongoDB, Redis, Kafka, and the other infrastructure components to be available.
:::

## Gateway

Once the gateway is running, it logs the following startup summary:

```text
Gateway Successfully Started On Port 3010
Swagger UI is running on: http://127.0.0.1:3010/api
Prometheus is running on: http://127.0.0.1:3010/metrics
Health check is running on: http://127.0.0.1:3010/status
OpenApi Spec is running on: http://127.0.0.1:3010/api-json
GraphQL playground is running on: http://127.0.0.1:3010/graphql
MCP streamable HTTP transport is running on: http://127.0.0.1:3010/mcp
```

### Health Check

The `/status` endpoint is the primary readiness signal. It verifies connectivity to Redis and confirms that every downstream gRPC provider is reachable:

```bash
curl http://127.0.0.1:3010/status
```

A fully healthy response:

```json
{
  "status": "ok",
  ...
}
```

Any entry with `"status": "down"` indicates that the corresponding service has not started or is unreachable from the gateway.

### Exposed Endpoints

| Endpoint | Purpose |
| --- | --- |
| `/status` | Readiness and dependency health check |
| `/api` | Swagger UI — interactive REST API explorer |
| `/api-json` | OpenAPI specification in JSON format — suitable for code generation and API client tooling |
| `/graphql` | GraphQL playground — schema introspection and query execution |
| `/mcp` | MCP streamable HTTP transport — entry point for AI agent tool calls |
| `/metrics` | Prometheus metrics — exposes request counts, latencies, and runtime statistics |
