# Docker Setup

Run the full platform (gateway, services, and workers) as Docker containers using the root `docker-compose.yml`. Each container uses the same image — the `SERVICE_NAME` environment variable selects which process to start.

## Prepare the Environment

Copy the environment template and generate a unique machine ID:

```bash
cp .env.example .env
npm run script:machine
```

`script:machine` writes a random `MACHINE_ID` into `.env` — required before starting any container.

## Build the Platform Image

From the repository root:

```bash
docker build -t wenex/platform:latest .
```

This runs `npm run git:clone`, `pnpm install --frozen-lockfile`, and `npm run script:build` inside the container, producing a self-contained image with all compiled services and workers under `wnx/`.

## Initialize the Database

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

## Start the Platform

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
