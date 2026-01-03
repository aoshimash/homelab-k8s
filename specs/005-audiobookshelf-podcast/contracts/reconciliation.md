# Contract: Flux Reconciliation for Audiobookshelf

**Feature**: 005-audiobookshelf-podcast
**Date**: 2026-01-03
**Type**: GitOps Reconciliation Contract

## Overview

This document defines the expected behavior when Flux CD reconciles the Audiobookshelf application from the Git repository.

## Reconciliation Flow

### Trigger Conditions

| Trigger | Description |
|---------|-------------|
| Git commit | Changes pushed to `k8s/apps/audiobookshelf/` |
| Interval | Every 10 minutes (default Flux interval) |
| Manual | `flux reconcile kustomization apps` |

### Dependency Chain

```
GitRepository (flux-system)
        │
        ▼
Kustomization (infrastructure)  ──dependsOn──▶  Kustomization (configs)
        │                                              │
        ▼                                              ▼
   [Longhorn]                                    [ProxyGroup]
   [Tailscale Operator]                          [Ingress configs]
                                                       │
                                                       ▼
                                              Kustomization (apps)
                                                       │
                                                       ▼
                                              [Audiobookshelf]
```

### Reconciliation Sequence

1. **Flux detects change** in `k8s/apps/` directory
2. **Dependency check**: Verify `configs` Kustomization is healthy
3. **Apply resources** in order:
   - Namespace (if not exists)
   - PersistentVolumeClaims
   - Deployment
   - Service
   - Ingress
4. **Wait for readiness**:
   - PVCs bound
   - Deployment available
   - Pods ready
5. **Report status** to Flux

## Expected Outcomes

### Success Criteria

| Resource | Expected State | Timeout |
|----------|----------------|---------|
| Namespace `audiobookshelf` | Active | 30s |
| PVC `audiobookshelf-config` | Bound | 2m |
| PVC `audiobookshelf-podcasts` | Bound | 2m |
| Deployment `audiobookshelf` | Available (1/1) | 5m |
| Service `audiobookshelf` | Created | 30s |
| Ingress `audiobookshelf` | Synced by Tailscale | 3m |
| Tailnet hostname | Accessible | 5m |

### Flux Kustomization Status

```yaml
# Expected status after successful reconciliation
status:
  conditions:
    - type: Ready
      status: "True"
      reason: ReconciliationSucceeded
      message: "Applied revision: main@sha1:xxxxxxx"
```

## Error Scenarios

### PVC Binding Failure

**Condition**: Longhorn unable to provision storage
**Symptoms**:
- PVC stuck in `Pending` state
- Events show provisioner errors
**Resolution**:
- Check Longhorn UI for storage availability
- Verify `longhorn` StorageClass exists

### Deployment Failure

**Condition**: Pod unable to start
**Symptoms**:
- Deployment shows 0/1 replicas
- Pod in `CrashLoopBackOff` or `ImagePullBackOff`
**Resolution**:
- Check pod events: `kubectl describe pod -n audiobookshelf`
- Verify image accessibility
- Check volume mounts

### Ingress Not Working

**Condition**: Service accessible internally but not via tailnet
**Symptoms**:
- Service responds to `kubectl port-forward`
- Tailscale hostname not resolving
**Resolution**:
- Check ProxyGroup status: `kubectl get proxygroup -n tailscale`
- Verify Tailscale Operator logs
- Confirm tailnet ACLs allow access

## Health Checks

### Kubernetes-level

```bash
# Check Flux Kustomization status
flux get kustomizations apps

# Check all resources
kubectl get all -n audiobookshelf

# Check PVC status
kubectl get pvc -n audiobookshelf

# Check Ingress status
kubectl get ingress -n audiobookshelf
```

### Application-level

```bash
# Direct health check (from within cluster)
kubectl exec -n audiobookshelf deploy/audiobookshelf -- wget -qO- localhost:80/healthcheck

# Via Tailscale (from tailnet device)
curl https://audiobookshelf.<tailnet>.ts.net/healthcheck
```

## Rollback Procedure

If reconciliation introduces issues:

1. **Identify last working commit**:
   ```bash
   git log --oneline k8s/apps/audiobookshelf/
   ```

2. **Revert to previous state**:
   ```bash
   git revert <bad-commit>
   git push
   ```

3. **Force reconciliation**:
   ```bash
   flux reconcile kustomization apps --with-source
   ```

4. **Verify recovery**:
   ```bash
   kubectl get all -n audiobookshelf
   ```

## Monitoring Integration

If Grafana Alloy is deployed:
- Pod metrics available in Grafana Cloud
- Logs from `audiobookshelf` namespace searchable in Loki
- Filter by: `{namespace="audiobookshelf"}`
