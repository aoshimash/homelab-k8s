# Data Model: Grafana Alloy for Metrics and Logs

**Feature**: 004-grafana-alloy
**Date**: 2026-01-03

## Kubernetes Resources

### 1. Namespace

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: monitoring
  labels:
    app.kubernetes.io/name: monitoring
```

**Purpose**: Dedicated namespace for observability infrastructure.

### 2. HelmRepository

```yaml
apiVersion: source.toolkit.fluxcd.io/v1
kind: HelmRepository
metadata:
  name: grafana
  namespace: flux-system
spec:
  interval: 1h
  url: https://grafana.github.io/helm-charts
```

**Purpose**: Source for Grafana Alloy Helm chart.

### 3. HelmRelease

```yaml
apiVersion: helm.toolkit.fluxcd.io/v2
kind: HelmRelease
metadata:
  name: grafana-alloy
  namespace: monitoring
spec:
  interval: 10m0s
  chart:
    spec:
      chart: alloy
      sourceRef:
        kind: HelmRepository
        name: grafana
        namespace: flux-system
      version: "0.x.x"  # Pin to stable version
  releaseName: grafana-alloy
  install:
    createNamespace: true
    remediation:
      retries: 3
  upgrade:
    remediation:
      retries: 3
  values:
    controller:
      type: daemonset
    alloy:
      configMap:
        content: |-
          // River configuration (see below)
  valuesFrom:
    - kind: Secret
      name: grafana-cloud-credentials
      valuesKey: GRAFANA_CLOUD_PROMETHEUS_URL
      targetPath: alloy.extraEnv[0].value
    # Additional env var mappings...
```

**Purpose**: Deploy Grafana Alloy as DaemonSet with configuration.

### 4. Secret (SOPS-encrypted)

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: grafana-cloud-credentials
  namespace: monitoring
type: Opaque
stringData:
  GRAFANA_CLOUD_PROMETHEUS_URL: "https://prometheus-prod-XX-XXX.grafana.net/api/prom/push"
  GRAFANA_CLOUD_LOKI_URL: "https://logs-prod-XX.grafana.net/loki/api/v1/push"
  GRAFANA_CLOUD_USER: "<instance-id>"
  GRAFANA_CLOUD_API_KEY: "<access-policy-token>"
```

**Purpose**: Store Grafana Cloud credentials encrypted in Git.

**Fields**:

| Field | Type | Description | Required |
|-------|------|-------------|----------|
| GRAFANA_CLOUD_PROMETHEUS_URL | string | Prometheus remote write endpoint URL | Yes |
| GRAFANA_CLOUD_LOKI_URL | string | Loki push endpoint URL | Yes |
| GRAFANA_CLOUD_USER | string | Grafana Cloud instance ID (numeric) | Yes |
| GRAFANA_CLOUD_API_KEY | string | Access Policy Token with metrics:write, logs:write | Yes |

### 5. Kustomization

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - namespace.yaml
  - helmrepository.yaml
  - helmrelease.yaml
  - secret-grafana-cloud.sops.yaml
```

**Purpose**: Bundle all Grafana Alloy resources for Flux reconciliation.

## Alloy Configuration (River)

### Core Components

```river
// =============================================================================
// LOGGING
// =============================================================================
logging {
  level  = "info"
  format = "logfmt"
}

// =============================================================================
// DISCOVERY
// =============================================================================

// Discover Kubernetes nodes
discovery.kubernetes "nodes" {
  role = "node"
}

// Discover Kubernetes pods
discovery.kubernetes "pods" {
  role = "pod"
}

// =============================================================================
// METRICS COLLECTION
// =============================================================================

// Scrape kubelet metrics (node-level)
prometheus.scrape "kubelet" {
  targets = discovery.kubernetes.nodes.targets
  
  job_name = "kubelet"
  scheme   = "https"
  
  tls_config {
    ca_file              = "/var/run/secrets/kubernetes.io/serviceaccount/ca.crt"
    insecure_skip_verify = true
  }
  bearer_token_file = "/var/run/secrets/kubernetes.io/serviceaccount/token"
  
  forward_to = [prometheus.relabel.add_cluster.receiver]
}

// Scrape cadvisor metrics (container-level)
prometheus.scrape "cadvisor" {
  targets = discovery.kubernetes.nodes.targets
  
  job_name     = "cadvisor"
  scheme       = "https"
  metrics_path = "/metrics/cadvisor"
  
  tls_config {
    ca_file              = "/var/run/secrets/kubernetes.io/serviceaccount/ca.crt"
    insecure_skip_verify = true
  }
  bearer_token_file = "/var/run/secrets/kubernetes.io/serviceaccount/token"
  
  forward_to = [prometheus.relabel.add_cluster.receiver]
}

// Add cluster label to all metrics
prometheus.relabel "add_cluster" {
  forward_to = [prometheus.remote_write.grafana_cloud.receiver]
  
  rule {
    target_label = "cluster"
    replacement  = "homelab"
  }
}

// Remote write to Grafana Cloud Prometheus
prometheus.remote_write "grafana_cloud" {
  endpoint {
    url = env("GRAFANA_CLOUD_PROMETHEUS_URL")
    
    basic_auth {
      username = env("GRAFANA_CLOUD_USER")
      password = env("GRAFANA_CLOUD_API_KEY")
    }
  }
  
  external_labels = {
    cluster = "homelab",
  }
}

// =============================================================================
// LOG COLLECTION
// =============================================================================

// Collect logs from all pods
loki.source.kubernetes "pods" {
  targets    = discovery.kubernetes.pods.targets
  forward_to = [loki.process.add_labels.receiver]
}

// Process and add labels
loki.process "add_labels" {
  forward_to = [loki.write.grafana_cloud.receiver]
  
  stage.static_labels {
    values = {
      cluster = "homelab",
    }
  }
}

// Write logs to Grafana Cloud Loki
loki.write "grafana_cloud" {
  endpoint {
    url = env("GRAFANA_CLOUD_LOKI_URL")
    
    basic_auth {
      username = env("GRAFANA_CLOUD_USER")
      password = env("GRAFANA_CLOUD_API_KEY")
    }
  }
  
  external_labels = {
    cluster = "homelab",
  }
}
```

## Data Flow

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           Kubernetes Cluster                                 │
│                                                                             │
│  ┌─────────────┐    ┌─────────────┐    ┌─────────────┐                     │
│  │   Node 1    │    │   Node 2    │    │   Node N    │                     │
│  │             │    │             │    │             │                     │
│  │ ┌─────────┐ │    │ ┌─────────┐ │    │ ┌─────────┐ │                     │
│  │ │ Alloy   │ │    │ │ Alloy   │ │    │ │ Alloy   │ │  (DaemonSet)       │
│  │ │ Pod     │ │    │ │ Pod     │ │    │ │ Pod     │ │                     │
│  │ └────┬────┘ │    │ └────┬────┘ │    │ └────┬────┘ │                     │
│  │      │      │    │      │      │    │      │      │                     │
│  │  ┌───┴───┐  │    │  ┌───┴───┐  │    │  ┌───┴───┐  │                     │
│  │  │kubelet│  │    │  │kubelet│  │    │  │kubelet│  │                     │
│  │  │metrics│  │    │  │metrics│  │    │  │metrics│  │                     │
│  │  │+logs  │  │    │  │+logs  │  │    │  │+logs  │  │                     │
│  │  └───────┘  │    │  └───────┘  │    │  └───────┘  │                     │
│  └─────────────┘    └─────────────┘    └─────────────┘                     │
│                                                                             │
└───────────────────────────────────┬─────────────────────────────────────────┘
                                    │
                                    │ HTTPS (Basic Auth)
                                    ▼
                    ┌───────────────────────────────────┐
                    │         Grafana Cloud             │
                    │                                   │
                    │  ┌─────────────┐ ┌─────────────┐ │
                    │  │ Prometheus  │ │    Loki     │ │
                    │  │ (Metrics)   │ │   (Logs)    │ │
                    │  └─────────────┘ └─────────────┘ │
                    │                                   │
                    │  ┌─────────────────────────────┐ │
                    │  │       Grafana UI            │ │
                    │  │   (Dashboards & Explore)    │ │
                    │  └─────────────────────────────┘ │
                    └───────────────────────────────────┘
```

## Labels and Metadata

### Metrics Labels

| Label | Source | Example |
|-------|--------|---------|
| `cluster` | Static (external_labels) | `homelab` |
| `node` | discovery.kubernetes | `talos-node-1` |
| `namespace` | discovery.kubernetes | `default` |
| `pod` | discovery.kubernetes | `nginx-deployment-abc123` |
| `container` | discovery.kubernetes | `nginx` |
| `job` | prometheus.scrape | `kubelet`, `cadvisor` |

### Log Labels

| Label | Source | Example |
|-------|--------|---------|
| `cluster` | Static (external_labels) | `homelab` |
| `namespace` | loki.source.kubernetes | `default` |
| `pod` | loki.source.kubernetes | `nginx-deployment-abc123` |
| `container` | loki.source.kubernetes | `nginx` |
| `node_name` | loki.source.kubernetes | `talos-node-1` |

## Resource Requirements (Estimated)

| Resource | Request | Limit | Notes |
|----------|---------|-------|-------|
| CPU | 100m | 500m | Per Alloy pod |
| Memory | 128Mi | 512Mi | Per Alloy pod; may need increase for high log volume |
| Storage | None | None | WAL stored in emptyDir |

## Validation Rules

1. **Secret must exist before HelmRelease**: Flux dependency ordering ensures secret is created first
2. **All credential fields required**: Missing fields will cause Alloy startup failure
3. **Valid URLs**: Endpoint URLs must be valid Grafana Cloud endpoints
4. **Valid API key**: Token must have `metrics:write` and `logs:write` scopes
