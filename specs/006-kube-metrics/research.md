# Research: kube-state-metrics

**Feature Branch**: `006-kube-metrics`  
**Date**: 2026-01-03

## Research Summary

This document captures research findings for implementing kube-state-metrics in the homelab cluster.

---

## 1. kube-state-metrics Helm Chart Selection

### Decision
Use the official prometheus-community/kube-state-metrics Helm chart.

### Rationale
- Official and actively maintained by the Prometheus community
- Latest version: 5.37.0 (released 2025-06-13)
- Wide adoption and well-documented
- Simple deployment model (single Deployment, not DaemonSet)

### Alternatives Considered
| Alternative | Rejected Because |
|-------------|------------------|
| Manual manifests | More maintenance overhead, no version management |
| Prometheus Operator bundle | Overkill for homelab, includes unnecessary components |
| Grafana Alloy built-in | Alloy doesn't generate kube-state-metrics, only scrapes them |

---

## 2. Namespace Strategy

### Decision
Deploy kube-state-metrics to the existing `monitoring` namespace.

### Rationale
- Grafana Alloy already runs in `monitoring` namespace
- Reduces namespace proliferation in small homelab cluster
- Simplifies service discovery (same namespace)
- Consistent with observability stack co-location pattern

### Alternatives Considered
| Alternative | Rejected Because |
|-------------|------------------|
| `kube-system` namespace | Mix of observability with system components |
| Dedicated `kube-state-metrics` namespace | Unnecessary namespace proliferation |

---

## 3. Grafana Alloy Integration Pattern

### Decision
Use Kubernetes service discovery with `role = "service"` to find kube-state-metrics endpoint, then scrape via `prometheus.scrape` component.

### Rationale
- Automatic endpoint discovery (no hardcoded IPs/ports)
- Consistent with how kubelet and cAdvisor are already scraped
- Service discovery handles pod restarts gracefully
- Follows Grafana Alloy best practices

### Implementation Pattern
```river
// Discover kube-state-metrics service
discovery.kubernetes "kube_state_metrics" {
  role = "service"
  namespaces {
    names = ["monitoring"]
  }
  selectors {
    role = "service"
    label = "app.kubernetes.io/name=kube-state-metrics"
  }
}

// Scrape kube-state-metrics
prometheus.scrape "kube_state_metrics" {
  targets    = discovery.kubernetes.kube_state_metrics.targets
  job_name   = "kube-state-metrics"
  forward_to = [prometheus.relabel.add_cluster.receiver]
}
```

### Alternatives Considered
| Alternative | Rejected Because |
|-------------|------------------|
| Static target configuration | Doesn't handle pod restarts, requires manual updates |
| Endpoint role discovery | Service role is simpler for single-service discovery |

---

## 4. Resource Limits

### Decision
Use Helm chart defaults initially, with option to tune if needed.

### Rationale
- kube-state-metrics is lightweight for small clusters
- Chart defaults are tuned for typical deployments
- Can be adjusted post-deployment based on actual usage
- Small homelab cluster (~100 objects) won't stress defaults

### Default Values (from chart)
- Requests: 10m CPU, 32Mi memory (typical)
- Limits: Unset by default (relies on namespace quotas if any)

### Monitoring Strategy
- Observe `kube_state_metrics_*` self-metrics after deployment
- Adjust if memory usage exceeds 100Mi for this cluster size

---

## 5. Metrics Exposed

### Decision
Use default metrics collection (all resource types enabled).

### Rationale
- Small cluster doesn't require metric filtering
- All default metrics are valuable for observability
- Filtering can be added later if cardinality becomes an issue

### Key Metrics for Verification
| Metric | Purpose |
|--------|---------|
| `kube_deployment_spec_replicas` | Deployment desired state |
| `kube_deployment_status_replicas_available` | Deployment actual state |
| `kube_pod_status_phase` | Pod lifecycle state |
| `kube_node_status_condition` | Node health conditions |
| `kube_persistentvolumeclaim_status_phase` | Storage state |

---

## 6. HelmRepository Configuration

### Decision
Add new HelmRepository resource for prometheus-community.

### Rationale
- prometheus-community is the official Helm chart repository
- Not currently configured in the cluster
- Reusable for future Prometheus ecosystem charts

### Configuration
```yaml
apiVersion: source.toolkit.fluxcd.io/v1
kind: HelmRepository
metadata:
  name: prometheus-community
  namespace: flux-system
spec:
  interval: 1h0m0s
  url: https://prometheus-community.github.io/helm-charts
```

---

## Open Questions (Resolved)

| Question | Resolution |
|----------|------------|
| Which Helm chart to use? | prometheus-community/kube-state-metrics v5.37.0 |
| Which namespace? | `monitoring` (existing) |
| How to integrate with Alloy? | Kubernetes service discovery + prometheus.scrape |
| Resource limits? | Use chart defaults, tune if needed |
| Metrics filtering? | No filtering, expose all defaults |
