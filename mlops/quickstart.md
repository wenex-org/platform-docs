---
title: "MLOps Quickstart"
description: "Get a MongoDB collection flowing into a LakeFS Delta Lake table: add a config.yaml source, write the script, deploy with Helm and verify."
---

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
  - name: my-grants             # unique name; its MD5 hash is the Celery task ID and the Redis key suffix
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

Add a `main()` function. Its contract — the keyword arguments it receives, the `pl.DataFrame` it
returns, the `id <= last_id` guard that keeps a batch from racing new writes, and the
`setting['latest_id']` it must record — is documented once, in
[Scripts → The `main()` function](./scripts#the-main-function), with the full reference
implementation. The skeleton is:

```python
import polars as pl

def main(**kwargs) -> pl.DataFrame:
    conn, setting = kwargs['conn'], kwargs['setting']   # the full kwarg list is in Scripts → Signature
    # 1. read the rows past setting['latest_id'] from PostgreSQL (`conn`), bounded by
    #    the MAX(id) taken at the start of the batch
    # 2. flatten each archived document into a row of a polars DataFrame
    # 3. record setting['latest_id'] = the last id you consumed
    # 4. return the DataFrame — the runner writes it to the LakeFS Delta table
    ...
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
helm repo add wenex-mlops https://vhidvz.github.io/charts  # the maintainer's chart host; `wenex` is the org host for the platform charts
helm repo update
helm upgrade --install mlops wenex-mlops/mlops -f values.yaml
```

## Step 4 — Verify

**Check Flower** — Open the Flower UI at `http://<flower-pod>:5555`. Within 5 minutes of the **`batch_size`-th** unprocessed row (100 in the config above — `db_check` queues a task only once at least `batch_size` rows sit past the last processed id; set `batch_size: 1` to see the first write), you should see a `script_runner` task appear with status `SUCCESS`.

**Check LakeFS** — Open the LakeFS UI and navigate to your repository. You should see new commits on the `main` branch and a Delta Lake table (`grants/` directory) in the file browser.

**Check PostgreSQL** — The archive table `auth.grants` will exist with rows. After the Worker processes them, `db_clean` (runs every 4 hours) purges rows up to the smallest `latest_id` any consumer of that table has recorded.

## What happens next

Once data flows in, the pipeline is self-maintaining:

- **Beat** checks for new data every 5 minutes and dispatches work automatically.
- **Workers** upsert on `id`, so re-running on the same data is safe.
- **Commits** happen on the configured interval (e.g. every 10 days).
- **Tags** create snapshots (e.g. every 30 days) and optionally trigger Airflow DAGs.

See [Architecture](./architecture) for a detailed explanation of the data flow.
