# Deployment

The MLOps system is deployed on Kubernetes using a Helm chart composed of three sub-charts: **Beat**, **Workers**, and **Flower**. All three share the same container image and environment configuration defined in the parent `values.yaml`.

## Helm Chart Repository

The MLOps Helm chart is published in the official Wenex chart repository:

```bash
helm repo add wenex https://wenex-org.github.io/charts
helm repo update
```

Verify the chart is available:

```bash
helm search repo wenex/mlops
```

### Chart Structure

```text
mlops/
├── Chart.yaml          # parent chart
├── values.yaml         # global config + sub-chart toggles
└── charts/
    ├── beat/           # Celery Beat scheduler (singleton)
    ├── flower/         # Celery monitoring UI
    └── workers/        # Celery task executors (scalable)
```

The parent chart also deploys the **Collector** pods (`main.py`) that consume Kafka CDC events and write to PostgreSQL.

## Sub-chart Roles

| Sub-chart | Default Replicas | Command | Notes |
| --- | --- | --- | --- |
| **beat** | 1 | `celery -A tasks beat --loglevel=info --schedule=.data/celerybeat-schedule` | Must always be exactly 1 replica. Multiple Beat instances cause duplicate task dispatches. |
| **flower** | 1 | `celery -A tasks flower` | Monitoring UI on port `5555`. |
| **workers** | 3 (configurable) | `celery -A tasks worker --loglevel=info` | Stateless; can be scaled horizontally. |

## Enabling Sub-charts

Sub-charts are disabled by default. Enable them in `values.yaml`:

```yaml
beat:
  enabled: true

flower:
  enabled: true

workers:
  enabled: true
```

## Environment Variables Reference

All environment variables are set under `global.envs` in `values.yaml` and are injected into every pod (Collector, Beat, Workers, Flower).

### LakeFS

| Variable | Description |
| --- | --- |
| `LAKEFS_SERVER_ENDPOINT_URL` | LakeFS server base URL (e.g. `https://lakefs.example.com`) |
| `LAKEFS_CREDENTIALS_ACCESS_KEY_ID` | LakeFS access key ID |
| `LAKEFS_CREDENTIALS_SECRET_ACCESS_KEY` | LakeFS secret access key |

### Kafka

| Variable | Description |
| --- | --- |
| `KAFKA_BOOTSTRAP_SERVERS` | Comma-separated broker addresses (e.g. `kafka-broker:9092`) |

### PostgreSQL

| Variable | Description |
| --- | --- |
| `POSTGRES_HOST` | Hostname of the PostgreSQL server |
| `POSTGRES_DB` | Database name (default: `lakefs`) |
| `POSTGRES_USER` | Database user |
| `POSTGRES_PASSWORD` | Database password |
| `POSTGRES_PORT` | Port (default: `5432`) |

### Redis

| Variable | Description |
| --- | --- |
| `REDIS_HOST` | Redis hostname |
| `REDIS_PORT` | Redis port (default: `6379`) |
| `REDIS_PASSWORD` | Redis password (leave empty if none) |
| `REDIS_PREFIX` | Key prefix for all MLOps Redis keys (default: `mlops`) |

### Celery

| Variable | Description |
| --- | --- |
| `CELERY_BROKER` | Celery broker URL (e.g. `redis://redis-host:6379/0`) |
| `CELERY_BACKEND` | Celery result backend URL (same host, different DB index is fine) |

### MongoDB

| Variable | Description |
| --- | --- |
| `MONGODB_URI` | Full MongoDB URI including auth and replica set (e.g. `mongodb://<POSTGRES_USER>:<POSTGRES_PASSWORD>@host/?replicaSet=rs0&authSource=admin`) |

## Example `values.yaml`

```yaml
replicaCount: 5        # collector pods

global:
  image:
    repository: registry.wenex.org/vhidvz/mlops
    pullPolicy: IfNotPresent
    tag: "1.0.0"
  envs:
    lakefs:
      serverEndpointUrl: "https://lakefs.wenex.org"
      credentialsAccessKeyId: "AKIAxxx"
      credentialsSecretAccessKey: "xxx"
    kafka:
      bootstrapServers: "wenex-kafka-bootstrap.kafka.svc.cluster.local:9092"
    postgres:
      host: "postgres-cluster-rw.cnpg-system.svc.cluster.local"
      port: "5432"
      db: "lakefs"
      user: <POSTGRES_USER>
      password: <POSTGRES_PASSWORD>
    redis:
      host: "redis-stack-master.redis.svc.cluster.local"
      port: "6379"
      password: <POSTGRES_PASSWORD>
      prefix: "mlops"
    celery:
      broker: "redis://redis-stack-master.redis.svc.cluster.local:6379/0"
      backend: "redis://redis-stack-master.redis.svc.cluster.local:6379/0"
    mongodb:
      uri: "mongodb://<POSTGRES_USER>:<POSTGRES_PASSWORD>@psmdb-server-psmdb-db-rs0.mongodb.svc.cluster.local/?replicaSet=rs0&authSource=admin"

beat:
  enabled: true

flower:
  enabled: true

workers:
  enabled: true
  replicaCount: 3    # override per sub-chart
```

Install or upgrade with your custom values:

```bash
helm upgrade --install mlops wenex/mlops -f values.yaml
```

## Scaling Workers

Increase or decrease Worker replicas by setting `workers.replicaCount` in `values.yaml`, or via `--set` at install time:

```bash
helm upgrade mlops wenex/mlops --set workers.replicaCount=6
```

Workers are stateless — scaling up adds capacity immediately without any coordination required.

**Do not scale Beat.** Beat stores its schedule state in `.data/celerybeat-schedule` and must run as a singleton. Running two Beat instances will result in duplicate task dispatches.

## Container Image

```text
registry.wenex.org/vhidvz/mlops:1.0.0
```

Base image: `apache/airflow:3.1.6-python3.12`. The image includes all dependencies (`celery`, `lakefs`, `deltalake`, `polars`, `pyarrow`, `psycopg2`, etc.) and the full `scripts/` directory.

## Accessing Flower

Flower runs on port `5555`. In a cluster environment, access it via port-forward:

```bash
kubectl port-forward svc/mlops-flower 5555:5555
```

Then open `http://localhost:5555` to view active workers, task history, queues, and execution times.
