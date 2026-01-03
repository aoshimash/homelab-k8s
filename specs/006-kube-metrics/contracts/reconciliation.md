# Contract: Flux Reconciliation for kube-state-metrics

**Feature Branch**: `006-kube-metrics`  
**Date**: 2026-01-03

## Overview

This document defines the expected behavior when Flux reconciles the kube-state-metrics resources.

---

## Preconditions

Before reconciliation:

1. **Flux System**: flux-system namespace exists and Flux controllers are running
2. **Monitoring Namespace**: `monitoring` namespace exists (created by Grafana Alloy)
3. **Git Repository**: kube-state-metrics manifests committed to `k8s/infrastructure/kube-state-metrics/`
4. **Infrastructure Kustomization**: `k8s/infrastructure/kustomization.yaml` includes `kube-state-metrics/`

---

## Reconciliation Sequence

### Phase 1: HelmRepository

**Trigger**: Flux Kustomize controller processes `helmrepository.yaml`

| Step | Action | Expected Result |
|------|--------|-----------------|
| 1 | Create HelmRepository | `prometheus-community` HelmRepository created in `flux-system` |
| 2 | Fetch index | Repository index downloaded from prometheus-community.github.io |
| 3 | Status update | HelmRepository shows `Ready: True` |

**Verification**:
```bash
kubectl get helmrepository prometheus-community -n flux-system
# Expected: Ready=True
```

### Phase 2: HelmRelease

**Trigger**: Flux Helm controller processes `helmrelease.yaml`

| Step | Action | Expected Result |
|------|--------|-----------------|
| 1 | Resolve chart | kube-state-metrics chart resolved from prometheus-community repo |
| 2 | Template chart | Kubernetes manifests generated from Helm chart |
| 3 | Apply manifests | Deployment, Service, ServiceAccount, ClusterRole, ClusterRoleBinding created |
| 4 | Wait for ready | Deployment reaches ready state |
| 5 | Status update | HelmRelease shows `Ready: True` |

**Verification**:
```bash
kubectl get helmrelease kube-state-metrics -n monitoring
# Expected: Ready=True

kubectl get deployment kube-state-metrics -n monitoring
# Expected: 1/1 Ready
```

### Phase 3: Grafana Alloy Config Update (Manual Step)

**Trigger**: Grafana Alloy HelmRelease updated with new scrape config

| Step | Action | Expected Result |
|------|--------|-----------------|
| 1 | Update HelmRelease | New scrape config added to Alloy configuration |
| 2 | Alloy pod restart | ConfigMap change triggers rolling restart |
| 3 | Service discovery | Alloy discovers kube-state-metrics service |
| 4 | Scrape start | Alloy begins scraping kube-state-metrics /metrics |
| 5 | Remote write | Metrics sent to Grafana Cloud Prometheus |

**Verification**:
```bash
kubectl logs -n monitoring -l app.kubernetes.io/name=alloy | grep kube-state-metrics
# Expected: Logs showing successful scrape
```

---

## Postconditions

After successful reconciliation:

1. **kube-state-metrics Pod**: Running in `monitoring` namespace
2. **kube-state-metrics Service**: ClusterIP service exposing port 8080
3. **Metrics Endpoint**: `/metrics` endpoint accessible within cluster
4. **Grafana Alloy**: Scraping kube-state-metrics and forwarding to Grafana Cloud
5. **Grafana Cloud**: `kube_*` metrics queryable with `cluster=homelab` label

---

## Error Scenarios

### HelmRepository Fetch Failure

**Cause**: Network issue or invalid URL  
**Symptom**: HelmRepository stuck in `Ready: False`  
**Resolution**: Check network connectivity, verify URL

```bash
kubectl describe helmrepository prometheus-community -n flux-system
```

### HelmRelease Install Failure

**Cause**: Invalid values, RBAC issues  
**Symptom**: HelmRelease stuck in `Ready: False`  
**Resolution**: Check Helm controller logs, review values

```bash
kubectl logs -n flux-system -l app=helm-controller
kubectl describe helmrelease kube-state-metrics -n monitoring
```

### Scrape Target Not Found

**Cause**: Service discovery misconfiguration  
**Symptom**: No `kube_*` metrics in Grafana Cloud  
**Resolution**: Verify service labels, check Alloy logs

```bash
kubectl get svc kube-state-metrics -n monitoring --show-labels
kubectl logs -n monitoring -l app.kubernetes.io/name=alloy
```

---

## Rollback Procedure

To remove kube-state-metrics:

1. Remove scrape config from Grafana Alloy HelmRelease
2. Remove `kube-state-metrics/` from `k8s/infrastructure/kustomization.yaml`
3. Commit and push changes
4. Flux automatically removes resources

**Note**: Historical metrics in Grafana Cloud remain until retention period expires.
