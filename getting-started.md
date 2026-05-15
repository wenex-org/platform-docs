# Getting Started

This guide walks you from a fresh clone to a fully running Wenex Platform.

---

## Prerequisites

| Requirement | Version |
|---|---|
| Node.js | 22.x (see `.nvmrc`) |
| pnpm | 10.5.2 |
| Docker + Docker Compose | 24+ |
| Git | any recent |

---

## 1. Clone and Install

```bash
git clone git@github.com:wenex-org/platform.git
cd platform

# Copy environment template
cp .env.example .env

# Pull git submodules (protos/)
npm run git:clone

# Install all dependencies
pnpm install --frozen-lockfile

# Generate protobuf TypeScript stubs + machine setup
npm run script:setup
```

---

## 2. Configure Environment

Edit `.env`. The key variables:

```env
# Gateway
GATEWAY_API_PORT=3010

# Per-service pattern: {SERVICE}_HOST, {SERVICE}_API_PORT, {SERVICE}_GRPC_PORT
AUTH_HOST=localhost
AUTH_API_PORT=3020
AUTH_GRPC_PORT=5020

# MongoDB
MONGO_HOST=localhost
MONGO_DB=wenex
MONGO_USER=root
MONGO_PASS=secret
MONGO_PREFIX=          # leave empty for prod; use "e2e" for E2E tests

# PostgreSQL (workers)
POSTGRES_HOST=localhost
POSTGRES_DB=wenex
POSTGRES_USER=root
POSTGRES_PASSWORD=secret

# Redis
REDIS_HOST=localhost
REDIS_PORT=6379
REDIS_PREFIX=

# Kafka
KAFKA_HOST=localhost
KAFKA_PORT=9092

# Elasticsearch
ELASTICSEARCH_NODE=http://localhost:9200

# MinIO
MINIO_HOST=localhost
MINIO_ACCESS_KEY=minioadmin
MINIO_SECRET_KEY=minioadmin

# EMQX (MQTT)
EMQX_BASE_URL=http://localhost:18083
EMQX_USERNAME=admin
EMQX_PASSWORD=public

# Observability
DEBUG=wnx:*
NODE_ENV=develop
```

---

## 3. Start Infrastructure

```bash
docker-compose -f docker/docker-compose.yml up -d
```

This starts: MongoDB, PostgreSQL, Redis, Kafka, Zookeeper, Elasticsearch, MinIO, EMQX.

Optional stacks (elk, observability, etc.):

```bash
docker-compose -f docker/docker-compose.elk.yml up -d
docker-compose -f docker/docker-compose.otlp.yml up -d
```

---

## 4. Seed Databases and Initialize Storage

```bash
# Seed MongoDB with initial data
npm run db:seed

# Create Elasticsearch indices
npm run db:index

# Initialize MinIO buckets
npm run storage:init

# Configure EMQX webhooks
npm run utility:init
```

---

## 5. Build the Project

```bash
# Build all apps in parallel (gateway → services → workers)
npm run script:build

# Or build a single project
npm run build auth
npm run build gateway
```

---

## 6. Start Services

### Option A — Docker (all-in-one)

```bash
docker build -t wenex/platform:latest .

# Start gateway + all services
docker-compose --profile services up -d

# Configure Kafka Connect (required before starting workers)
docker-compose --profile kafka-connect up

# Start workers
docker-compose --profile workers up -d
```

### Option B — Manual (watch mode for development)

Start each service in a separate terminal. The recommended startup order:

```bash
# Core services first
npm run start:dev auth
npm run start:dev domain
npm run start:dev context
npm run start:dev essential
npm run start:dev identity

# Domain services
npm run start:dev financial
npm run start:dev special
npm run start:dev career
npm run start:dev touch
npm run start:dev content
npm run start:dev logistic
npm run start:dev conjoint
npm run start:dev general
npm run start:dev thing

# Gateway last
npm run start:dev gateway

# Configure Kafka Connect
npm run script:kafka-connect

# Workers
npm run start:dev dispatcher
npm run start:dev observer
npm run start:dev preserver
npm run start:dev watcher
npm run start:dev publisher
npm run start:dev logger
npm run start:dev cleaner
```

With debug:

```bash
npm run start:debug auth      # Debug on default port
npm run start:debug2 auth     # Debug on alternate port
```

---

## 7. Verify the Platform

```bash
# Gateway health check
curl http://localhost:3010/status

# Swagger UI
open http://localhost:3010/api

# GraphQL Playground
open http://localhost:3010/graphql

# Prometheus metrics
curl http://localhost:3010/metrics
```

---

## E2E Test Environment

E2E tests use prefixed databases (`e2e_` prefix) so they don't pollute production data.

```bash
# Seed E2E databases
npm run db:seed:e2e
npm run db:index:e2e
npm run storage:init
npm run utility:init

# Start services in E2E mode (in one terminal)
npm run script:start:e2e auth domain context essential identity special gateway

# Configure Kafka Connect for E2E
npm run script:kafka-connect:e2e

# Start workers in E2E mode (in another terminal)
npm run script:start:e2e dispatcher observer preserver watcher publisher logger cleaner

# Run tests
npm run test:e2e
```

---

## Code Quality

```bash
# Format all files with Prettier
npm run format

# Lint and auto-fix with ESLint
npm run lint

# Check for circular dependencies
npm run check

# Generate circular dependency graph image
npm run madge:report:ts
```

---

## Port Reference

| Component | HTTP/REST | gRPC |
|---|---|---|
| Gateway | 3010 | — |
| Auth service | 3020 | 5020 |
| Domain service | 3030 | 5030 |
| Context service | 3040 | 5040 |
| Essential service | 3050 | 5050 |
| Financial service | 3060 | 5060 |
| General service | 3070 | 5070 |
| Identity service | 3080 | 5080 |
| Special service | 3090 | 5090 |
| Touch service | 3100 | 5100 |
| Content service | 3110 | 5110 |
| Logistic service | 3120 | 5120 |
| Conjoint service | 3130 | 5130 |
| Career service | 3140 | 5140 |
| Thing service | 3150 | 5150 |
| Dispatcher worker | 4010 | — |
| Observer worker | 4020 | — |
| Preserver worker | 4030 | — |
| Watcher worker | 4040 | — |
| Publisher worker | 4050 | — |
| Logger worker | 4060 | — |
| Cleaner worker | 4070 | — |
