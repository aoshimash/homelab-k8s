# Reconciliation Contract: Grafana Alloy

**Feature**: 004-grafana-alloy
**Date**: 2026-01-03

## Overview

This contract defines the expected behavior when Flux reconciles Grafana Alloy resources from Git to the cluster.

## Preconditions

Before reconciliation can succeed:

1. **Flux system running**: `flux-system` namespace exists with source-controller and kustomize-controller
2. **SOPS decryption configured**: `sops-age` secret exists in `flux-system` namespace
3. **Git repository accessible**: Flux can fetch from the configured GitRepository
4. **Grafana Cloud credentials**: Valid Access Policy Token with `metrics:write` and `logs:write` scopes
5. **Network access**: Cluster can reach `grafana.net` endpoints (HTTPS/443)

## Reconciliation Flow

```
┌─────────────────────────────────────────────────────────────────────────┐
│                        Flux Reconciliation                              │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  1. GitRepository sync                                                  │
│     └─► Fetch latest commit from main branch                           │
│                                                                         │
│  2. Kustomization: infrastructure                                       │
│     ├─► Path: ./k8s/infrastructure                                     │
│     ├─► SOPS decryption enabled                                        │
│     └─► Includes: grafana-alloy/                                       │
│                                                                         │
│  3. Resource creation order (Kustomize)                                │
│     ├─► Namespace: monitoring                                          │
│     ├─► Secret: grafana-cloud-credentials (decrypted)                  │
│     ├─► HelmRepository: grafana                                        │
│     └─► HelmRelease: grafana-alloy                                     │
│                                                                         │
│  4. Helm controller                                                    │
│     ├─► Fetch chart from grafana helm repo                            │
│     ├─► Render with values (including secret refs)                    │
│     └─► Apply: DaemonSet, ConfigMap, ServiceAccount, RBAC             │
│                                                                         │
│  5. DaemonSet rollout                                                  │
│     └─► One Alloy pod per node                                         │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

## Expected Outcomes

### Success State

| Resource | Namespace | Expected Status |
|----------|-----------|-----------------|
| Namespace | `monitoring` | Active |
| Secret | `monitoring` | Created (decrypted) |
| HelmRepository | `flux-system` | Ready |
| HelmRelease | `monitoring` | Ready, revision applied |
| DaemonSet | `monitoring` | Available, desired = ready |
| Pods | `monitoring` | Running on each node |

### Verification Commands

```bash
# Check Flux Kustomization status
flux get kustomizations infrastructure

# Check HelmRelease status
flux get helmreleases -n monitoring

# Check DaemonSet status
kubectl get daemonset -n monitoring grafana-alloy

# Check pod logs for successful startup
kubectl logs -n monitoring -l app.kubernetes.io/name=alloy --tail=50

# Verify metrics are being sent (look for remote_write success)
kubectl logs -n monitoring -l app.kubernetes.io/name=alloy | grep -i "remote_write"

# Verify logs are being sent
kubectl logs -n monitoring -l app.kubernetes.io/name=alloy | grep -i "loki"
```

### Grafana Cloud Verification

```bash
# In Grafana Cloud Explore (Prometheus):
# Query: up{cluster="homelab"}
# Expected: Returns metrics from cluster nodes

# In Grafana Cloud Explore (Loki):
# Query: {cluster="homelab"}
# Expected: Returns logs from cluster pods
```

## Failure Scenarios

### 1. SOPS Decryption Failure

**Symptoms**:
- Kustomization shows `ReconciliationFailed`
- Secret not created or contains encrypted data

**Resolution**:
```bash
# Verify sops-age secret exists
kubectl get secret -n flux-system sops-age

# Check Kustomization events
kubectl describe kustomization infrastructure -n flux-system
```

### 2. Helm Chart Fetch Failure

**Symptoms**:
- HelmRepository shows `FetchFailed`
- HelmRelease stuck in `Progressing`

**Resolution**:
```bash
# Check HelmRepository status
kubectl describe helmrepository grafana -n flux-system

# Verify network connectivity
kubectl run test-curl --rm -it --restart=Never --image=curlimages/curl -- \
  curl -s https://grafana.github.io/helm-charts/index.yaml | head
```

### 3. Invalid Grafana Cloud Credentials

**Symptoms**:
- Alloy pods running but logs show authentication errors
- No data appearing in Grafana Cloud

**Resolution**:
```bash
# Check Alloy pod logs for auth errors
kubectl logs -n monitoring -l app.kubernetes.io/name=alloy | grep -i "401\|403\|auth"

# Verify secret contents (base64 decode)
kubectl get secret -n monitoring grafana-cloud-credentials -o jsonpath='{.data.GRAFANA_CLOUD_USER}' | base64 -d
```

### 4. River Configuration Syntax Error

**Symptoms**:
- Alloy pods in CrashLoopBackOff
- Logs show config parsing errors

**Resolution**:
```bash
# Check pod logs for config errors
kubectl logs -n monitoring -l app.kubernetes.io/name=alloy --previous

# Validate River syntax locally (if alloy binary available)
alloy fmt config.alloy
```

### 5. Network Connectivity Issues

**Symptoms**:
- Alloy running but remote_write failing
- Logs show connection timeout/refused

**Resolution**:
```bash
# Test connectivity to Grafana Cloud endpoints
kubectl run test-curl --rm -it --restart=Never --image=curlimages/curl -- \
  curl -sv https://prometheus-prod-XX-XXX.grafana.net/api/prom/push 2>&1 | head -20

# Check if egress is blocked
kubectl get networkpolicies -A
```

## Rollback Procedure

If reconciliation causes issues:

```bash
# Option 1: Suspend reconciliation and investigate
flux suspend kustomization infrastructure

# Option 2: Revert Git commit and let Flux reconcile
git revert HEAD
git push

# Option 3: Manual deletion (last resort)
kubectl delete helmrelease -n monitoring grafana-alloy
kubectl delete namespace monitoring
```

## Reconciliation Interval

| Resource | Interval | Notes |
|----------|----------|-------|
| GitRepository | 1m | Source polling |
| Kustomization (infrastructure) | 10m | Applied changes |
| HelmRepository | 1h | Chart updates |
| HelmRelease | 10m | Drift correction |

## Dependencies

```
GitRepository (flux-system)
    │
    └─► Kustomization (infrastructure)
            │
            ├─► Namespace (monitoring)
            │
            ├─► Secret (grafana-cloud-credentials)
            │
            ├─► HelmRepository (grafana)
            │       │
            │       └─► HelmRelease (grafana-alloy)
            │               │
            │               └─► DaemonSet, ConfigMap, RBAC
            │
            └─► [Other infrastructure components]
```

## Monitoring the Reconciliation

### Flux Events

```bash
# Watch Flux events
flux events --watch

# Get all Flux resources status
flux get all -A
```

### Alerting (Optional)

Configure Flux notification controller to alert on reconciliation failures:

```yaml
apiVersion: notification.toolkit.fluxcd.io/v1beta3
kind: Alert
metadata:
  name: infrastructure-alerts
  namespace: flux-system
spec:
  providerRef:
    name: slack  # or other provider
  eventSeverity: error
  eventSources:
    - kind: Kustomization
      name: infrastructure
    - kind: HelmRelease
      name: grafana-alloy
      namespace: monitoring
```
