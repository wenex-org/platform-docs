# Kubernetes Setup

Deploy the full platform on Kubernetes using the official Helm chart published to [Artifact Hub](https://artifacthub.io/packages/helm/wenex/platform).

## Using Helm Charts

```bash
helm repo add wenex https://wenex-org.github.io/charts
helm repo update
```

See the [Configuration](./configuration) section for the full list of available values and their equivalent `.env` variables.

## Install the Chart

```bash
helm install platform wenex/platform
```

To customize values inline:

```bash
helm install platform wenex/platform \
  --set global.environments.nodeEnv=production \
  --set global.environments.mongo.pass=secret
```

Or pass a values file:

```bash
helm install platform wenex/platform -f my-values.yaml
```

## Control Subcharts

By default, all subcharts are disabled. Enable the ones you need under the top-level keys:

```yaml
# values.yaml
auth:
  enabled: true
identity:
  enabled: true
gateway:
  enabled: true
```

The full list of available keys mirrors the service and worker names: `auth`, `domain`, `context`, `essential`, `identity`, `financial`, `career`, `special`, `touch`, `content`, `logistic`, `conjoint`, `general`, `thing`, `watcher`, `observer`, `preserver`, `dispatcher`, `publisher`, `logger`, `cleaner`.
