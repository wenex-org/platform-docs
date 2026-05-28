# Start Infrastructure

```bash
docker-compose -f docker/docker-compose.yml up -d
```

This starts the full core stack: MongoDB (replica set), PostgreSQL, Redis, Kafka, Zookeeper, Elasticsearch, MinIO, and EMQX.

## Additional Compose

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

## MongoDB Replica Set

The main compose file runs MongoDB as a replica set. Add these entries to `/etc/hosts` so the hostnames resolve correctly:

```text
127.0.0.1 mongodb-primary
127.0.0.1 mongodb-secondary
```

Connection URL:

```text
mongodb://root:password123@mongodb-primary:27017,mongodb-secondary:27018/?replicaSet=rs0&authSource=admin
```
