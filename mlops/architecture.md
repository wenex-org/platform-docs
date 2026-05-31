# MLOps Architecture

The MLOps system connects the Wenex platform's operational data to a versioned ML-ready data lake. It captures every MongoDB document change through Kafka CDC, archives it in PostgreSQL, and periodically transforms it into Delta Lake tables in LakeFS via configurable Python scripts.

## Components

| Component | Entry Point | Role |
| --- | --- | --- |
| **Collector** | `main.py` | Kafka consumer — subscribes to `mongo.wnx-*` CDC topics and writes `before`/`after` JSON payloads to PostgreSQL archive tables |
| **Beat** | `tasks.db_check` | Celery scheduler — runs every 5 minutes to check if any script's row count has reached its `batch_size` threshold, then queues a `script_runner` task |
| **Beat** | `tasks.db_clean` | Celery scheduler — runs every 4 hours to drop orphaned PG tables, remove stale Redis keys, and delete already-processed archive rows |
| **Workers** | `tasks.script_runner` | Celery workers — dynamically load and execute `scripts/<name>/__main__.py`, upsert the returned Polars DataFrame into LakeFS Delta Lake, and manage commit/tag intervals |
| **Flower** | `celery flower` | Web UI at port `5555` for monitoring Celery task queues, worker status, and task history |

## Storage Layers

| Store | Purpose | Schema / Key Pattern |
| --- | --- | --- |
| **PostgreSQL** | CDC archive — rows written by the Collector wait here until a Worker processes them | Table per collection: `id SERIAL, oid VARCHAR, after JSONB, before JSONB` |
| **Redis** | Script execution state — tracks the last processed row ID and the next scheduled commit/tag timestamps | `script:{md5(name)}` → JSON: `{latest_id, tag_interval, commit_interval}` |
| **LakeFS** | Versioned Delta Lake storage — the destination for transformed DataFrames; supports branching, committing, and tagging | S3-compatible: `s3://{repository_id}/{branch}/{delta_table}` |
| **MongoDB** | Source of truth — the platform's operational collections; also accessible to scripts via the `client` parameter | Native Atlas/PSMDB replica set |

## Data Flow

```mermaid
sequenceDiagram
    participant MDB as MongoDB
    participant KFK as Kafka
    participant COL as Collector (main.py)
    participant PG as PostgreSQL
    participant BEAT as Celery Beat
    participant WRK as Worker
    participant RDS as Redis
    participant LFS as LakeFS
    participant AIR as Airflow

    MDB->>KFK: CDC event on change (mongo.wnx-* topic)
    KFK->>COL: consume message
    COL->>PG: INSERT (oid, after, before) into archive table

    loop every 5 minutes
        BEAT->>PG: SELECT MAX(id), COUNT(*) for each script source
        BEAT->>RDS: GET latest_id for script
        alt (last_id - latest_id) >= batch_size
            BEAT->>WRK: apply_async(script_runner, name=script_name)
        end
    end

    WRK->>PG: query archive rows WHERE id > latest_id
    WRK->>WRK: execute scripts/<name>/__main__.py → Polars DataFrame
    WRK->>LFS: merge DataFrame into Delta Lake table (upsert on id)
    WRK->>RDS: SET latest_id, check commit/tag intervals
    alt commit_interval reached
        WRK->>LFS: branch.commit(timestamp)
    end
    alt tag_interval reached
        WRK->>LFS: branch.tag(timestamp).create()
        LFS->>AIR: POST webhook → trigger Airflow DAG
    end
```

### Step-by-step

1. **CDC capture** — MongoDB emits a change event; the Kafka CDC connector publishes it to a topic named `mongo.wnx-<db>.<collection>`.
2. **Archive** — The Collector (`main.py`) consumes the event and inserts a row into a PostgreSQL table named after the collection (e.g. `auth.grants`), storing the full `after` and `before` document JSON.
3. **Threshold check** — Beat's `db_check` task queries each archive table. It compares `last_id - latest_id` against the script's `batch_size`. If the threshold is met and no task is already running for that script, it dispatches a `script_runner` task using the script's MD5 hash as the Celery task ID (ensuring uniqueness).
4. **Script execution** — A Worker dynamically imports `scripts/<name>/__main__.py` and calls its `main()` function, which queries the archive table and returns a Polars DataFrame.
5. **Delta Lake upsert** — The Worker merges the DataFrame into `s3://{repository_id}/{branch}/{delta_table}` using Delta Lake's `merge` with `source.id = target.id` as the predicate (insert-or-update semantics). On first run it creates the table.
6. **Versioning** — After every successful write, the Worker checks Redis for scheduled commit and tag timestamps. When the intervals expire, it commits and/or tags the LakeFS branch.
7. **DAG trigger** — A LakeFS action (registered in the repository) fires an HTTP POST to Airflow when a new tag is created, triggering downstream ML workflows.

## Concurrency Model

- **Beat is a singleton** — only one Beat pod must run at a time. Running multiple Beat instances will cause duplicate task dispatches.
- **Workers are stateless** — they can be scaled horizontally. Each `script_runner` task uses the script's MD5 hash as its Celery task ID, so Beat will not queue a second run for the same script while one is already `PENDING` or running.
- **No concurrent script execution** — Before dispatching, Beat checks `AsyncResult(script_hash).state`. If the state is not `SUCCESS` (or forgotten), it skips dispatch. This prevents overlapping runs of the same script even with multiple Workers.

## Full Architecture Diagram

```mermaid
graph TB
    subgraph Platform["Wenex Platform"]
        MDB["MongoDB\n(Replica Set)"]
        KFK["Kafka Broker"]
        KFK_CON["Kafka CDC Connector\nmongo.wnx-* topics"]
    end

    subgraph MLOps["MLOps Kubernetes Deployment"]
        COL["Collectors\n(Pods × 5)\nmain.py"]
        BEAT["Beat\n(Pod × 1)\nCelery Scheduler"]
        WRK["Workers\n(Pods × 3)\nCelery Executors"]
        FLW["Flower\n(Pod × 1)\nport 5555"]
    end

    subgraph Storage["Data Stores"]
        PG["PostgreSQL\nCDC Archive"]
        RDS["Redis\nState & Broker"]
        LFS["LakeFS\nDelta Lake"]
        MDB2["MongoDB\n(script access)"]
    end

    subgraph Downstream["Downstream ML"]
        AIR["Apache Airflow\nDAG Orchestration"]
        MLFL["MLflow\nExperiment Tracking"]
    end

    MDB -->|CDC| KFK
    KFK --> KFK_CON --> COL
    COL -->|archive rows| PG

    BEAT -->|read state| RDS
    BEAT -->|check thresholds| PG
    BEAT -->|dispatch tasks| RDS

    WRK -->|read archive| PG
    WRK -->|read/write state| RDS
    WRK -->|upsert DataFrame| LFS
    WRK -->|read documents| MDB2

    FLW -->|monitor| RDS

    LFS -->|tag webhook| AIR
    AIR -->|read Delta Lake| LFS
    AIR --> MLFL
```
