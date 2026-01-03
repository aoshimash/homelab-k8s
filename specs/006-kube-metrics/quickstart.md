# Quickstart: kube-state-metrics

**Feature Branch**: `006-kube-metrics`  
**Date**: 2026-01-03

## Prerequisites

- [ ] Flux CD running in cluster
- [ ] Grafana Alloy deployed in `monitoring` namespace
- [ ] Grafana Cloud credentials configured
- [ ] Access to homelab-k8s Git repository

## Quick Deploy Steps

### Step 1: Create kube-state-metrics Directory

```bash
mkdir -p k8s/infrastructure/kube-state-metrics
```

### Step 2: Add HelmRepository

Create `k8s/infrastructure/kube-state-metrics/helmrepository.yaml`:

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

### Step 3: Add HelmRelease

Create `k8s/infrastructure/kube-state-metrics/helmrelease.yaml`:

```yaml
apiVersion: helm.toolkit.fluxcd.io/v2
kind: HelmRelease
metadata:
  name: kube-state-metrics
  namespace: monitoring
spec:
  interval: 10m0s
  chart:
    spec:
      chart: kube-state-metrics
      sourceRef:
        kind: HelmRepository
        name: prometheus-community
        namespace: flux-system
  releaseName: kube-state-metrics
  install:
    createNamespace: false
    remediation:
      retries: 3
  upgrade:
    remediation:
      retries: 3
```

### Step 4: Add Kustomization

Create `k8s/infrastructure/kube-state-metrics/kustomization.yaml`:

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - helmrepository.yaml
  - helmrelease.yaml
```

### Step 5: Update Infrastructure Kustomization

Edit `k8s/infrastructure/kustomization.yaml` to add:

```yaml
resources:
  - cilium/
  - longhorn/
  - tailscale/
  - grafana-alloy/
  - kube-state-metrics/  # Add this line
```

### Step 6: Update Grafana Alloy Config

Add scrape config to `k8s/infrastructure/grafana-alloy/helmrelease.yaml`:

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

### Step 7: Commit and Push

```bash
git add k8s/infrastructure/kube-state-metrics/
git add k8s/infrastructure/kustomization.yaml
git add k8s/infrastructure/grafana-alloy/helmrelease.yaml
git commit -m "feat(infra): add kube-state-metrics for Kubernetes resource state monitoring"
git push origin 006-kube-metrics
```

### Step 8: Verify Deployment

```bash
# Check HelmRepository
kubectl get helmrepository prometheus-community -n flux-system

# Check HelmRelease
kubectl get helmrelease kube-state-metrics -n monitoring

# Check Pod
kubectl get pods -n monitoring -l app.kubernetes.io/name=kube-state-metrics

# Check metrics endpoint
kubectl port-forward -n monitoring svc/kube-state-metrics 8080:8080
curl localhost:8080/metrics | head -20
```

### Step 9: Verify in Grafana Cloud

Query in Grafana Cloud Prometheus:

```promql
kube_deployment_spec_replicas{cluster="homelab"}
```

Expected: List of deployments with their desired replica counts.

## Troubleshooting

### Pod not starting

```bash
kubectl describe pod -n monitoring -l app.kubernetes.io/name=kube-state-metrics
kubectl logs -n monitoring -l app.kubernetes.io/name=kube-state-metrics
```

### No metrics in Grafana Cloud

1. Check Grafana Alloy is scraping:
   ```bash
   kubectl logs -n monitoring -l app.kubernetes.io/name=alloy | grep kube-state
   ```

2. Check service discovery:
   ```bash
   kubectl get svc kube-state-metrics -n monitoring --show-labels
   ```

### HelmRelease stuck

```bash
kubectl describe helmrelease kube-state-metrics -n monitoring
flux reconcile helmrelease kube-state-metrics -n monitoring
```
