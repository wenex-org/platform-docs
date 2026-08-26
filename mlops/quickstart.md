# Quickstart

This guide walks through the minimum steps to get a MongoDB collection flowing into a LakeFS Delta Lake table using the MLOps pipeline.

## Prerequisites

- A running Wenex platform with at least one service writing to MongoDB
- Kafka broker with CDC connectors enabled (`mongo.wnx-*` topics available)
- PostgreSQL database accessible by the MLOps system
- Redis instance
- LakeFS server (with an access key and secret key)
- Helm 3+ (for Kubernetes deployment)

## Step 1 — Add a source to `config.yaml`

Open `scripts/config.yaml` and add an entry under `scripts:`.

```yaml
scripts:
  - name: my-grants             # unique name; becomes the Celery task ID prefix
    branch: main                # LakeFS branch to write to
    delta_table: grants         # Delta Lake table name inside the repository
    repository_id: my-repo      # LakeFS repository (created automatically if absent)
    storage_namespace: s3://lakefs/my-repo  # storage backend for the repository
    source: auth.grants         # MongoDB collection: <service>.<collection>
    batch_size: 100             # queue a task when 100+ unprocessed rows exist
    tag_interval: 30 days       # create a LakeFS tag every 30 days
    commit_interval: 10 days    # commit the branch every 10 days
    condition: 'after IS NOT NULL'  # optional: only process insert/update events
```

The `source` field must match the collection name as it appears in the PostgreSQL archive table — formatted as `<db>.<collection>` (e.g. `auth.grants`, `domain.users`).

## Step 2 — Create the script

Create the directory and entry point for your script:

```bash
mkdir -p scripts/my-grants
touch scripts/my-grants/__main__.py
```

Add a `main()` function. The simplest possible script that flattens each archived document into a row:

```python
import polars as pl


def batch(rows: list) -> pl.DataFrame:
    columns = ['id', 'oid', 'after', 'before']
    column_data = dict(zip(columns, zip(*rows)))
    column_data.pop('id')
    column_data['id'] = column_data.pop('oid')
    return pl.DataFrame(column_data)


def main(**kwargs):
    conn = kwargs['conn']
    script = kwargs['script']
    setting = kwargs['setting']

    latest_id = setting.get('latest_id', 0)

    with conn.cursor() as cur:
        condition = f"WHERE id > {latest_id}"
        if 'condition' in script:
            condition = f"WHERE {script['condition']} AND id > {latest_id}"

        cur.execute(f"""
            SELECT MAX(id), COUNT(*)
            FROM "{script['source']}" {condition};
        """)
        last_id, total_count = cur.fetchone()

        if total_count == 0:
            return pl.DataFrame()

        cur.execute(f"""
            SELECT * FROM "{script['source']}" {condition}
            ORDER BY id ASC LIMIT 10000;
        """)

        df = pl.DataFrame()
        while True:
            rows = cur.fetchmany(script['batch_size'])
            if not rows:
                break
            df = pl.concat([df, batch(rows)])

    setting['latest_id'] = last_id
    return df
```

See [Scripts](./scripts) for the full `main()` parameter reference and how the LakeFS lifecycle works.

## Step 3 — Deploy with Helm

Enable the sub-charts and configure your environment in `values.yaml`:

```yaml
global:
  envs:
    lakefs:
      serverEndpointUrl: "https://lakefs.example.com"
      credentialsAccessKeyId: "your-access-key"
      credentialsSecretAccessKey: "your-secret-key"
    kafka:
      bootstrapServers: "kafka-broker:9092"
    postgres:
      host: "postgres-host"
      db: "lakefs"
      user: "lakefs"
      password: "lakefs"
    redis:
      host: "redis-host"
    celery:
      broker: "redis://redis-host:6379/0"
      backend: "redis://redis-host:6379/0"
    mongodb:
      uri: "mongodb://user:pass@mongo-host/?replicaSet=rs0"

beat:
  enabled: true

flower:
  enabled: true

workers:
  enabled: true
```

Add the Wenex chart repository and install:

```bash
helm repo add wenex https://vhidvz.github.io/charts
helm repo update
helm upgrade --install mlops wenex/mlops -f values.yaml
```

## Step 4 — Verify

**Check Flower** — Open the Flower UI at `http://<flower-pod>:5555`. Within 5 minutes of your first MongoDB write, you should see a `script_runner` task appear with status `SUCCESS`.

**Check LakeFS** — Open the LakeFS UI and navigate to your repository. You should see new commits on the `main` branch and a Delta Lake table (`grants/` directory) in the file browser.

**Check PostgreSQL** — The archive table `auth.grants` will exist with rows. After the Worker processes them, `db_clean` (runs every 4 hours) will purge consumed rows.

## What happens next

Once data flows in, the pipeline is self-maintaining:

- **Beat** checks for new data every 5 minutes and dispatches work automatically.
- **Workers** upsert on `id`, so re-running on the same data is safe.
- **Commits** happen on the configured interval (e.g. every 10 days).
- **Tags** create snapshots (e.g. every 30 days) and optionally trigger Airflow DAGs.

See [Architecture](./architecture) for a detailed explanation of the data flow.
