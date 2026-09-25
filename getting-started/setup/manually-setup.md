---
title: "Manual Setup from Source"
description: "Run the Wenex Platform from source: clone and install, initialize the databases, set up Kafka Connect and start the platform."
---

# Manually Setup

Run the platform from source. Follow each step in order.

## Clone and Install

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

## Initialization

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

## Kafka Connect

This step is required before starting platform services. It registers a [Debezium](https://debezium.io) MongoDB source connector with Kafka Connect, enabling Change Data Capture (CDC) across all platform collections.

The connector monitors every write to the MongoDB databases (auth, identity, domain, context, financial, career, conjoint, content, logistic, general, special, touch, thing, and essential sagas) and publishes change events to Kafka under the `mongo.` topic prefix. Workers such as `dispatcher`, `observer`, and `publisher` consume these events to drive notifications, audit logs, webhooks, and saga orchestration.

Run after the infrastructure and Kafka Connect container are up:

```sh
npm run script:kafka-connect
```

If the connector already exists it is automatically removed and re-registered, so this command is safe to re-run.

## Start Platform

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
| `essential` | Distributed saga orchestration with MongoDB-backed saga state and compensating steps |
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

**Gateway** — the public entry point that exposes REST, GraphQL, and MCP (`/mcp`) and calls the services above over gRPC:

| Project | Description |
| --- | --- |
| `gateway` | Starts the unified gateway (default port: `3010`) |

**Workers** — background processes with no public REST API; all but `preserver` and `cleaner` consume Kafka CDC events:

| Project | Description |
| --- | --- |
| `dispatcher` | Delivers CQRS webhooks: posts each change event to the webhook URL of every subscribed client, stashing failures in PostgreSQL for BullMQ retries |
| `observer` | Keeps create/update/delete counters per owner in `special/stats` from change events |
| `preserver` | The EMQX ExHook gRPC server: authenticates and authorizes MQTT clients and topic access |
| `watcher` | Caches grants, APTs, OAuth clients, users and CQRS configs in Redis; indexes products, messages and posts in Elasticsearch |
| `publisher` | Publishes an MQTT notification through EMQX for each change, on topics derived from the document's owner, shares, groups and clients |
| `logger` | Persists the audit log events all services emit to PostgreSQL via TypeORM |
| `cleaner` | Purges records older than their retention TTL — audit and stash logs (PostgreSQL), stats, metrics and saga stages (MongoDB) |
