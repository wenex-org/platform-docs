# Configuration

All platform configuration is expressed as environment variables. For Docker and manual deployments these live in a `.env` file; for Kubernetes they are declared under `global` in `values.yaml` and injected into pods as environment variables at runtime.

Each subsection below covers one configuration group. Every table lists the `values.yaml` path (Helm), its `.env` counterpart, the default value, and a description of what the variable controls.

## Secrets

Cryptographic secrets used across the platform for authentication, encryption, and hashing. All five values are **auto-generated** at install time if left empty — a random string of the appropriate length is produced and stored in a Kubernetes `Secret` (or written to `.env` for local setups). In production, supply explicit values and store them in a secrets manager to ensure consistency across restarts and replicas.

| `.env` variable | `values.yaml` path | Default | Description |
| --- | --- | --- | --- |
| `ROOT_PASSWORD` | `global.secrets.init.rootPassword` | random 40 chars | Password for the root admin account seeded during `db:seed`. |
| `CLIENT_SECRET` | `global.secrets.init.clientSecret` | random 40 chars | Secret for the default OAuth client created during initialization. |
| `AES_SECRET` | `global.secrets.aes` | random 64 chars | Key used for AES-256 symmetric encryption of sensitive fields stored in the database. |
| `JWT_SECRET` | `global.secrets.jwt` | random 64 chars | Signing key for all JWT access and refresh tokens issued by the `auth` service. |
| `BCRYPT_SALT` | `global.secrets.bcrypt` | random 32 chars | Salt used by bcrypt for password hashing. |

## General

Runtime behavior settings that apply globally to every service and worker process.

| `.env` variable | `values.yaml` path | Default | Description |
| --- | --- | --- | --- |
| `NODE_ENV` | `global.environments.nodeEnv` | `develop` | Runtime environment. Set to `production` for deployed instances — affects logging verbosity, error detail, and compiler optimizations. |
| `DEBUG` | `global.environments.debug` | `wnx:*` | Namespace filter for the `debug` module. `wnx:*` enables all platform debug output; set to an empty string to silence it. |
| `TIMEOUT` | `global.environments.timeout` | `90000` | Global timeout in milliseconds applied to outbound gRPC calls between the gateway and services. |

## Internationalization

Controls the default locale and timezone applied to date formatting, number formatting, and time-based operations across all services.

| `.env` variable | `values.yaml` path | Default | Description |
| --- | --- | --- | --- |
| `LOCALE` | `global.environments.locale` | `fa` | BCP 47 language tag for the default locale (e.g., `fa` for Persian, `en` for English). |
| `REGION` | `global.environments.region` | `IR` | ISO 3166-1 alpha-2 country code used for locale-specific number and currency formatting. |
| `TZ` | `global.environments.tz` | `Asia/Tehran` | IANA timezone identifier applied to all date/time operations across the platform. |

## Redis

Redis is used for distributed caching, session storage, rate limiting, and BullMQ job queues. All services and workers share the same Redis instance, isolated from each other via the key prefix.

| `.env` variable | `values.yaml` path | Default | Description |
| --- | --- | --- | --- |
| `REDIS_HOST` | `global.environments.redis.host` | `redis-stack-master.redis.svc.cluster.local` | Hostname of the Redis instance. |
| `REDIS_PORT` | `global.environments.redis.port` | `6379` | Port Redis listens on. |
| `REDIS_PREFIX` | `global.environments.redis.prefix` | `wnx` | Key prefix applied to every Redis entry to prevent collisions in shared instances. |
| `REDIS_PASSWORD` | `global.environments.redis.password` | | Authentication password. Leave empty if Redis runs without auth. |

## MinIO

MinIO provides S3-compatible object storage for the `special` service, which handles file uploads. Two buckets — `public` and `private` — are created during the `storage:init` step.

| `.env` variable | `values.yaml` path | Default | Description |
| --- | --- | --- | --- |
| `MINIO_HOST` | `global.environments.minio.host` | `minio.minio-tenant.svc.cluster.local` | Hostname of the MinIO server. |
| `MINIO_PORT` | `global.environments.minio.port` | `80` | HTTP port MinIO listens on (typically `80` in-cluster, `9000` for direct access). |
| `MINIO_ACCESS_KEY` | `global.environments.minio.accessKey` | | MinIO access key ID used for authentication. |
| `MINIO_SECRET_KEY` | `global.environments.minio.secretKey` | | MinIO secret access key used for authentication. |

## MongoDB

MongoDB is the primary data store for all 14 domain services. The platform requires a replica set — change data capture via Debezium depends on the oplog, which is only available in replica set mode.

| `.env` variable | `values.yaml` path | Default | Description |
| --- | --- | --- | --- |
| `MONGO_HOST` | `global.environments.mongo.host` | `psmdb-server-psmdb-db-rs0.mongodb.svc.cluster.local` | Comma-separated list of `host:port` pairs for the replica set members. |
| `MONGO_DB` | `global.environments.mongo.db` | `wenex` | Name of the primary MongoDB database. |
| `MONGO_PREFIX` | `global.environments.mongo.prefix` | `wnx` | Prefix applied to all collection names, enabling isolation in a shared database. |
| `MONGO_USER` | `global.environments.mongo.user` | `databaseAdmin` | MongoDB username with read/write access to the target database. |
| `MONGO_PASS` | `global.environments.mongo.pass` | | MongoDB password. |
| `MONGO_QUERY` | `global.environments.mongo.query` | `replicaSet=rs0&loadBalanced=true&authSource=admin` | Additional query string parameters appended to the MongoDB connection URI. |

## PostgreSQL

PostgreSQL is used exclusively by the `essential` service for durable, transactional saga state. It stores the compensating steps required to roll back distributed transactions safely.

| `.env` variable | `values.yaml` path | Default | Description |
| --- | --- | --- | --- |
| `POSTGRES_HOST` | `global.environments.postgres.host` | `postgres-cluster-rw.cnpg-system.svc.cluster.local` | Hostname of the PostgreSQL server. |
| `POSTGRES_PORT` | `global.environments.postgres.port` | `5432` | Port PostgreSQL listens on. |
| `POSTGRES_DB` | `global.environments.postgres.db` | `wenex` | Name of the PostgreSQL database. |
| `POSTGRES_PREFIX` | `global.environments.postgres.prefix` | `wnx` | Prefix applied to table names to avoid collisions in a shared database. |
| `POSTGRES_USER` | `global.environments.postgres.user` | `postgres` | PostgreSQL username. |
| `POSTGRES_PASSWORD` | `global.environments.postgres.password` | | PostgreSQL password. |

## Elasticsearch

Elasticsearch stores aggregated logs shipped by the `logger` worker, full-text search indices for messages, posts, and products (maintained by `preserver`), and is also used by the APM stack.

| `.env` variable | `values.yaml` path | Default | Description |
| --- | --- | --- | --- |
| `ELASTICSEARCH_NODE` | `global.environments.elasticsearch.node` | `https://elk-cluster-es-http.elastic-system.svc.cluster.local:9200` | Full URL (protocol + host + port) of the Elasticsearch node. |
| `ELASTIC_PREFIX` | `global.environments.elasticsearch.prefix` | `wnx` | Prefix applied to all index names created by the platform. |
| `ELASTICSEARCH_USERNAME` | `global.environments.elasticsearch.username` | `elastic` | Elasticsearch username for Basic Auth. |
| `ELASTICSEARCH_PASSWORD` | `global.environments.elasticsearch.password` | | Elasticsearch password for Basic Auth. |

## Kafka

Kafka is the event bus that carries CDC change events from MongoDB (via Debezium) to the worker layer. All workers — `dispatcher`, `observer`, `preserver`, `publisher`, `logger`, and `watcher` — consume topics under the `mongo.` prefix.

| `.env` variable | `values.yaml` path | Default | Description |
| --- | --- | --- | --- |
| `KAFKA_HOST` | `global.environments.kafka.host` | `wenex-kafka-bootstrap.kafka.svc.cluster.local` | Hostname of the Kafka broker. |
| `KAFKA_PORT` | `global.environments.kafka.port` | `9092` | Port the Kafka broker listens on. |
| `KAFKAJS_NO_PARTITIONER_WARNING` | `global.environments.kafka.noPartitionerWarning` | `1` | Set to `1` to suppress KafkaJS default partitioner deprecation warnings. |

## EMQX

EMQX is the MQTT broker used by the `conjoint` service for real-time messaging. The platform registers an ExHook gRPC server (hosted by the `preserver` worker) that EMQX calls to authorize publish and subscribe actions before delivering messages.

| `.env` variable | `values.yaml` path | Default | Description |
| --- | --- | --- | --- |
| `EMQX_USERNAME` | `global.environments.emqx.username` | `admin` | Admin username for the EMQX HTTP management API. |
| `EMQX_PASSWORD` | `global.environments.emqx.password` | | Admin password for the EMQX HTTP management API. |
| `EMQX_EXHOOK_URL` | `global.environments.emqx.exhookUrl` | `http://platform-preserver.wenex-platform.svc.cluster.local:8080` | URL of the platform's gRPC ExHook server that EMQX calls for publish/subscribe authorization. |
| `EMQX_BASE_URL` | `global.environments.emqx.baseUrl` | `http://emqx-dashboard.emqx-operator-system.svc.cluster.local:18083/api/v5` | Base URL of the EMQX REST API used to manage rules, connections, and authorization sources. |

## OpenStreetMap

Optional geospatial services used by the `logistic` domain. Both can be self-hosted or replaced with any compatible API endpoint.

| `.env` variable | `values.yaml` path | Default | Description |
| --- | --- | --- | --- |
| `VALHALLA_API_BASE_URL` | `global.environments.valhalla.apiBaseUrl` | `http://valhalla.openstreetmap.svc.cluster.local` | Base URL of the Valhalla routing engine, used for turn-by-turn navigation and route calculation. |
| `NOMINATIM_API_BASE_URL` | `global.environments.nominatim.apiBaseUrl` | `http://mediagis-nominatim.openstreetmap.svc.cluster.local` | Base URL of the Nominatim geocoding service, used for address lookup and reverse geocoding. |

## TURN / STUN

Coturn provides WebRTC relay for NAT traversal, required by the `conjoint` service for peer-to-peer media streams. Time-limited credentials are generated using the shared secret so that no long-lived passwords are distributed to clients.

| `.env` variable | `values.yaml` path | Default | Description |
| --- | --- | --- | --- |
| `COTURN_AUTH_SECRET` | `global.environments.coturn.authSecret` | | Shared secret used to generate time-limited TURN credentials for WebRTC clients. |
| `COTURN_ICE_SERVERS` | `global.environments.coturn.iceServers` | `["turn.wenex.org"]` | List of TURN/STUN server addresses exposed to WebRTC peers via the ICE negotiation. |

## OpenTelemetry

Distributed traces and metrics are exported via the OpenTelemetry Protocol (OTLP) over HTTP. Any OTLP-compatible collector is supported — the default configuration targets a Jaeger instance deployed in the cluster.

| `.env` variable | `values.yaml` path | Default | Description |
| --- | --- | --- | --- |
| `OTLP_HOST` | `global.environments.otlp.host` | `jaeger-instance-collector.opentelemetry.svc.cluster.local` | Hostname of the OTLP-compatible collector that receives traces and metrics. |
| `OTLP_PORT` | `global.environments.otlp.port` | `4318` | HTTP port the collector listens on for the `http/protobuf` OTLP exporter. |

## Elastic APM

Application Performance Monitoring via the Elastic APM agent. When configured, every service reports transaction timings, error rates, and slow spans to the APM server for analysis in Kibana.

| `.env` variable | `values.yaml` path | Default | Description |
| --- | --- | --- | --- |
| `ELASTIC_APM_SERVER_URL` | `global.environments.apm.serverUrl` | | URL of the Elastic APM server that ingests performance data. |
| `ELASTIC_APM_SECRET_TOKEN` | `global.environments.apm.secretToken` | | Secret token used to authenticate with the APM server. |
| `ELASTIC_APM_VERIFY_SERVER_CERT` | `global.environments.apm.verifyServerCert` | `false` | Whether to verify the APM server's TLS certificate. Set to `false` for self-signed certificates. |

## Sentry

Error tracking via Sentry. When a DSN is provided, unhandled exceptions and rejected promises from every service and worker are captured and reported, along with request breadcrumbs and performance traces.

| `.env` variable | `values.yaml` path | Default | Description |
| --- | --- | --- | --- |
| `SENTRY_DSN` | `global.environments.sentry.dsn` | | Data Source Name URL for the Sentry project that receives error reports. Leave empty to disable. |
| `SENTRY_MAX_BREADCRUMBS` | `global.environments.sentry.maxBreadcrumbs` | `100` | Maximum number of breadcrumb entries captured per error event. |
| `SENTRY_TRACES_SAMPLE_RATE` | `global.environments.sentry.tracesSampleRate` | `0.8` | Fraction of transactions sampled for Sentry performance monitoring (0.0–1.0). |

## Cleaner Worker

The `cleaner` worker runs on a schedule and hard-deletes soft-deleted records once their retention period has elapsed. Each TTL value accepts a human-readable duration string (e.g., `30day`, `6months`, `2years`).

| `.env` variable | `values.yaml` path | Default | Description |
| --- | --- | --- | --- |
| `CLEANER_STATS_TTL` | `global.environments.cleaner.statsTTL` | `10years` | Retention period for aggregated statistics records. |
| `CLEANER_METRICS_TTL` | `global.environments.cleaner.metricsTTL` | `5years` | Retention period for time-series metrics records. |
| `CLEANER_AUDIT_LOGS_TTL` | `global.environments.cleaner.auditLogsTTL` | `4years` | Retention period for audit log entries tracking user and system actions. |
| `CLEANER_STASH_LOGS_TTL` | `global.environments.cleaner.stashLogsTTL` | `100day` | Retention period for stash/debug log entries. |
| `CLEANER_SAGA_STAGES_TTL` | `global.environments.cleaner.sagaStagesTTL` | `1year` | Retention period for completed saga stage records after a distributed transaction finishes. |
