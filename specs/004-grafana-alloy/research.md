# Research: Grafana Alloy for Metrics and Logs

**Feature**: 004-grafana-alloy
**Date**: 2026-01-03

## 1. Grafana Alloy Helm Chart Configuration

### Decision
Use official Grafana Alloy Helm chart from `https://grafana.github.io/helm-charts` with DaemonSet controller type.

### Rationale
- Official chart maintained by Grafana Labs
- Supports DaemonSet deployment mode required for log collection
- Configuration embedded in `values.yaml` under `alloy.configMap.content`
- Helm chart handles RBAC, ServiceAccount, and necessary permissions

### Alternatives Considered
1. **kubernetes-monitoring Helm chart**: More comprehensive but includes Grafana, Prometheus, etc. Overkill for sending to Grafana Cloud.
2. **Manual manifests**: More control but harder to maintain; Helm chart provides tested defaults.

### Key Configuration

```yaml
# HelmRelease values structure
controller:
  type: daemonset

alloy:
  configMap:
    content: |-
      // River configuration here
```

## 2. Alloy River Configuration for Kubernetes

### Decision
Use Grafana Alloy's River configuration language with the following components:
- `discovery.kubernetes` for target discovery
- `prometheus.scrape` for metrics collection
- `loki.source.kubernetes` for log collection
- `prometheus.remote_write` and `loki.write` for sending to Grafana Cloud

### Rationale
- River is the native configuration language for Alloy (successor to Grafana Agent Flow)
- Components are designed for Kubernetes-native discovery
- Built-in relabeling and processing pipelines

### Key Components

#### Metrics Discovery and Scraping

```river
// Discover Kubernetes pods
discovery.kubernetes "pods" {
  role = "pod"
}

// Discover Kubernetes nodes (for kubelet/cadvisor)
discovery.kubernetes "nodes" {
  role = "node"
}

// Scrape kubelet metrics
prometheus.scrape "kubelet" {
  targets    = discovery.kubernetes.nodes.targets
  scheme     = "https"
  tls_config {
    insecure_skip_verify = true
  }
  bearer_token_file = "/var/run/secrets/kubernetes.io/serviceaccount/token"
  forward_to = [prometheus.remote_write.grafana_cloud.receiver]
}

// Add cluster label to all metrics
prometheus.relabel "add_cluster_label" {
  forward_to = [prometheus.remote_write.grafana_cloud.receiver]
  rule {
    target_label = "cluster"
    replacement  = "homelab"
  }
}
```

#### Log Collection

```river
// Collect pod logs
loki.source.kubernetes "pods" {
  targets    = discovery.kubernetes.pods.targets
  forward_to = [loki.process.add_labels.receiver]
}

// Add cluster label to logs
loki.process "add_labels" {
  forward_to = [loki.write.grafana_cloud.receiver]
  stage.static_labels {
    values = {
      cluster = "homelab",
    }
  }
}
```

## 3. Grafana Cloud Authentication

### Decision
Use Basic Authentication with Grafana Cloud Access Policy Token.

### Rationale
- Grafana Cloud remote write endpoints support Basic Auth
- Access Policy Tokens can be scoped to specific permissions (metrics:write, logs:write)
- Token rotation is straightforward via Grafana Cloud console

### Required Credentials

| Credential | Description | Source |
|------------|-------------|--------|
| GRAFANA_CLOUD_PROMETHEUS_URL | Prometheus remote write endpoint | Grafana Cloud → Connections → Prometheus |
| GRAFANA_CLOUD_LOKI_URL | Loki push endpoint | Grafana Cloud → Connections → Loki |
| GRAFANA_CLOUD_USER | Instance ID (numeric) | Grafana Cloud → Account Settings |
| GRAFANA_CLOUD_API_KEY | Access Policy Token | Grafana Cloud → Access Policies |

### Access Policy Token Scopes
- `metrics:write` - Required for Prometheus remote write
- `logs:write` - Required for Loki push

## 4. Kubernetes State Metrics

### Decision
Use Grafana Alloy's built-in Kubernetes discovery and scraping capabilities rather than deploying separate kube-state-metrics.

### Rationale
- Alloy can discover and expose Kubernetes object metrics through `discovery.kubernetes`
- Reduces operational complexity (one less component to manage)
- For advanced kube-state-metrics, can add later if needed

### Alternative
If more detailed Kubernetes state metrics are needed:
- Deploy kube-state-metrics as a separate Deployment
- Configure Alloy to scrape its `/metrics` endpoint

## 5. Secret Management with SOPS

### Decision
Store Grafana Cloud credentials in a SOPS-encrypted Kubernetes Secret (`secret-grafana-cloud.sops.yaml`).

### Rationale
- Consistent with existing patterns (Longhorn R2 credentials, Tailscale OAuth)
- Flux decryption already configured for `k8s/infrastructure/`
- No plaintext secrets in Git

### Secret Structure

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: grafana-cloud-credentials
  namespace: monitoring
type: Opaque
stringData:
  GRAFANA_CLOUD_PROMETHEUS_URL: "https://prometheus-prod-XX-prod-XX-XXX.grafana.net/api/prom/push"
  GRAFANA_CLOUD_LOKI_URL: "https://logs-prod-XX.grafana.net/loki/api/v1/push"
  GRAFANA_CLOUD_USER: "123456"
  GRAFANA_CLOUD_API_KEY: "glc_..."
```

### Alloy Configuration Reference

```river
prometheus.remote_write "grafana_cloud" {
  endpoint {
    url = env("GRAFANA_CLOUD_PROMETHEUS_URL")
    basic_auth {
      username = env("GRAFANA_CLOUD_USER")
      password = env("GRAFANA_CLOUD_API_KEY")
    }
  }
}
```

## 6. Namespace Selection

### Decision
Deploy Grafana Alloy in `monitoring` namespace.

### Rationale
- Standard namespace for observability tools
- Separates monitoring infrastructure from application workloads
- Allows future addition of related tools (Prometheus, Grafana if needed locally)

## 7. Log Collection Scope

### Decision
Collect logs from all namespaces without exclusions.

### Rationale
- Homelab cluster is small; no need for filtering
- Complete visibility into all workloads
- Can add exclusion rules later if volume becomes an issue

### Potential Future Enhancement
If log volume is too high, add filtering:

```river
loki.process "filter" {
  forward_to = [loki.write.grafana_cloud.receiver]
  
  // Drop logs from noisy namespaces
  stage.match {
    selector = "{namespace=\"kube-system\"}"
    action   = "drop"
  }
}
```

## 8. Resilience and Buffering

### Decision
Rely on Alloy's built-in write-ahead log (WAL) for buffering during connectivity issues.

### Rationale
- Alloy includes WAL for prometheus.remote_write
- Automatic retry on failed sends
- No additional configuration needed for basic resilience

### Configuration (if explicit WAL needed)

```river
prometheus.remote_write "grafana_cloud" {
  endpoint {
    url = env("GRAFANA_CLOUD_PROMETHEUS_URL")
    // ...
  }
  wal {
    truncate_frequency = "2h"
    max_keepalive_time = "8h"
  }
}
```

## Summary of Technical Decisions

| Topic | Decision |
|-------|----------|
| Deployment | Grafana Alloy Helm chart, DaemonSet mode |
| Configuration | River language embedded in HelmRelease values |
| Metrics | discovery.kubernetes + prometheus.scrape for node/pod metrics |
| Logs | loki.source.kubernetes for container logs |
| Authentication | Basic Auth with Access Policy Token |
| Secrets | SOPS-encrypted Secret in Git |
| Namespace | `monitoring` |
| Cluster label | `cluster=homelab` added to all metrics/logs |
| State metrics | Built-in Kubernetes discovery (no separate kube-state-metrics) |
