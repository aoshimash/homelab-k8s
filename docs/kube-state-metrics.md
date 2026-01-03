# kube-state-metrics for Kubernetes Resource State Monitoring

**Feature**: 006-kube-metrics  
**Date**: 2026-01-03

## Overview

kube-state-metrics is deployed to collect Kubernetes resource state metrics (Deployments, Pods, Nodes, Services, etc.) and expose them via Prometheus-compatible `/metrics` endpoint. Grafana Alloy scrapes these metrics and forwards them to Grafana Cloud Prometheus, providing visibility into Kubernetes object states beyond CPU and memory usage.

## Architecture

- **Deployment**: Single-replica Deployment (not DaemonSet)
- **Namespace**: `monitoring` (shared with Grafana Alloy)
- **Metrics Source**: Kubernetes API server (watches resource states)
- **Metrics Endpoint**: `/metrics` on port 8080
- **Scraper**: Grafana Alloy (via Kubernetes service discovery)
- **Destination**: Grafana Cloud Prometheus (via Grafana Alloy)
- **Cluster Label**: `cluster=homelab` added by Grafana Alloy relabeling

## Prerequisites

- Kubernetes cluster running with Flux CD
- Grafana Alloy deployed in `monitoring` namespace
- Grafana Cloud credentials configured (via Grafana Alloy)

## Deployment

kube-state-metrics is deployed via GitOps using Flux HelmRelease:

- **HelmRepository**: `prometheus-community` (flux-system namespace)
- **HelmRelease**: `kube-state-metrics` (monitoring namespace)
- **Chart**: `prometheus-community/kube-state-metrics`

### GitOps Structure

```
k8s/infrastructure/kube-state-metrics/
├── helmrepository.yaml    # prometheus-community Helm repo
├── helmrelease.yaml      # kube-state-metrics HelmRelease
└── kustomization.yaml    # Kustomize resource list
```

## Metrics Exposed

kube-state-metrics generates metrics for the following Kubernetes resources:

| Resource Type | Metric Prefix | Example Metrics |
|---------------|---------------|-----------------|
| Deployments | `kube_deployment_*` | `spec_replicas`, `status_replicas_available`, `status_replicas_updated` |
| StatefulSets | `kube_statefulset_*` | `spec_replicas`, `status_replicas_ready` |
| DaemonSets | `kube_daemonset_*` | `status_desired_number_scheduled`, `status_number_ready` |
| ReplicaSets | `kube_replicaset_*` | `spec_replicas`, `status_ready_replicas` |
| Pods | `kube_pod_*` | `status_phase`, `container_status_ready`, `container_resource_limits` |
| Nodes | `kube_node_*` | `status_condition`, `status_capacity`, `info` |
| Services | `kube_service_*` | `info`, `spec_type` |
| PersistentVolumes | `kube_pv_*` | `status_phase`, `capacity_bytes` |
| PersistentVolumeClaims | `kube_pvc_*` | `status_phase`, `resource_requests_storage_bytes` |
| Namespaces | `kube_namespace_*` | `status_phase`, `labels` |
| Jobs | `kube_job_*` | `status_succeeded`, `status_failed`, `complete` |
| CronJobs | `kube_cronjob_*` | `status_active`, `next_schedule_time` |

All metrics include labels: `namespace`, `pod`, `node`, `cluster=homelab`, `job=kube-state-metrics`

## Access

### Metrics Endpoint (Cluster Internal)

```bash
# Port-forward to access metrics endpoint
kubectl port-forward -n monitoring svc/kube-state-metrics 8080:8080

# Query metrics
curl localhost:8080/metrics | grep kube_deployment
```

### Grafana Cloud Prometheus

Query kube-state-metrics in Grafana Cloud:

```promql
# Deployment replicas
kube_deployment_spec_replicas{cluster="homelab"}

# Pod phases
kube_pod_status_phase{cluster="homelab"}

# Node conditions
kube_node_status_condition{cluster="homelab"}
```

## Operations

### Check Deployment Status

```bash
# Check HelmRelease
kubectl get helmrelease kube-state-metrics -n monitoring

# Check Pod
kubectl get pods -n monitoring -l app.kubernetes.io/name=kube-state-metrics

# Check Service
kubectl get svc kube-state-metrics -n monitoring
```

### View Logs

```bash
kubectl logs -n monitoring -l app.kubernetes.io/name=kube-state-metrics
```

### Verify Metrics Collection

```bash
# Check Grafana Alloy is scraping kube-state-metrics
kubectl logs -n monitoring -l app.kubernetes.io/name=alloy | grep kube-state-metrics

# Check service discovery
kubectl get svc kube-state-metrics -n monitoring --show-labels
```

## Troubleshooting

### Pod Not Starting

**Symptoms**: Pod stuck in Pending or CrashLoopBackOff

**Diagnosis**:
```bash
kubectl describe pod -n monitoring -l app.kubernetes.io/name=kube-state-metrics
kubectl logs -n monitoring -l app.kubernetes.io/name=kube-state-metrics
```

**Common Issues**:
- RBAC permissions: Check ClusterRole and ClusterRoleBinding
- Resource limits: Check if pod is being evicted
- API server connectivity: Verify kube-state-metrics can reach Kubernetes API

### No Metrics in Grafana Cloud

**Symptoms**: `kube_*` metrics not appearing in Grafana Cloud

**Diagnosis**:
1. Verify kube-state-metrics is running:
   ```bash
   kubectl get pods -n monitoring -l app.kubernetes.io/name=kube-state-metrics
   ```

2. Verify Grafana Alloy is scraping:
   ```bash
   kubectl logs -n monitoring -l app.kubernetes.io/name=alloy | grep kube-state
   ```

3. Verify service discovery:
   ```bash
   kubectl get svc kube-state-metrics -n monitoring --show-labels
   # Should have: app.kubernetes.io/name=kube-state-metrics
   ```

**Common Issues**:
- Service labels missing: Ensure Helm chart sets correct labels
- Alloy config incorrect: Verify service discovery selectors match service labels
- Network policy blocking: Check if network policies allow Alloy → kube-state-metrics communication

### HelmRelease Stuck

**Symptoms**: HelmRelease shows `Ready: False`

**Diagnosis**:
```bash
kubectl describe helmrelease kube-state-metrics -n monitoring
kubectl logs -n flux-system -l app=helm-controller | grep kube-state-metrics
```

**Common Issues**:
- HelmRepository not ready: Check `kubectl get helmrepository prometheus-community -n flux-system`
- Chart version unavailable: Verify chart exists in prometheus-community repo
- Values validation error: Check HelmRelease values syntax

**Resolution**:
```bash
# Force reconciliation
flux reconcile helmrelease kube-state-metrics -n monitoring
```

## Backup and Restore

### Backup

kube-state-metrics is stateless - no persistent data to backup. Configuration is stored in Git:

```bash
# Backup manifests
git archive --format=tar.gz HEAD k8s/infrastructure/kube-state-metrics/ > kube-state-metrics-backup.tar.gz
```

### Restore

Restore from Git:

```bash
# Restore manifests
git checkout HEAD -- k8s/infrastructure/kube-state-metrics/
# Flux will automatically reconcile
```

## Upgrades

kube-state-metrics is upgraded automatically via Flux when the Helm chart is updated in the prometheus-community repository.

To pin a specific version, update `helmrelease.yaml`:

```yaml
spec:
  chart:
    spec:
      chart: kube-state-metrics
      version: "5.37.0"  # Pin version
```

## Monitoring

### Key Metrics to Monitor

- `kube_state_metrics_build_info`: Version information
- `kube_state_metrics_list_total`: Total number of list operations
- `kube_state_metrics_watch_total`: Total number of watch operations
- `kube_state_metrics_shard_*`: Shard-specific metrics (if sharding enabled)

### Resource Usage

Monitor kube-state-metrics resource consumption:

```bash
kubectl top pod -n monitoring -l app.kubernetes.io/name=kube-state-metrics
```

Typical resource usage for small homelab cluster:
- CPU: 10-50m
- Memory: 50-150Mi

## Related Documentation

- [Grafana Alloy Documentation](./grafana-alloy.md)
- [Flux GitOps Documentation](./flux-bootstrap.md)
- [kube-state-metrics Official Docs](https://github.com/kubernetes/kube-state-metrics)
