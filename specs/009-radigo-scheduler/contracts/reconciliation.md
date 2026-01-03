# Contract: Flux Reconciliation

## Overview

Radigo scheduler components are deployed via Flux GitOps reconciliation from the `k8s/apps/radigo/` directory.

## Kustomization Dependency Chain

```
flux-system
    └── apps-kustomization
            └── k8s/apps/radigo/kustomization.yaml
                    ├── namespace.yaml
                    ├── configmap-schedules.yaml
                    ├── secret-audiobookshelf-api.sops.yaml
                    ├── cronjob-arco.yaml
                    ├── cronjob-ijuin.yaml
                    └── (additional cronjobs)
```

## Resource Dependencies

### Prerequisites (must exist before radigo deployment)

| Resource | Namespace | Purpose |
|----------|-----------|---------|
| PVC `audiobookshelf-podcasts` | audiobookshelf | Shared storage for recordings |
| Service `audiobookshelf` | audiobookshelf | API endpoint for library scan |

### Radigo Resources

| Resource | Type | Purpose |
|----------|------|---------|
| Namespace `radigo` | Namespace | Isolation for radigo components |
| ConfigMap `radigo-schedules` | ConfigMap | Program schedule configuration |
| Secret `radigo-audiobookshelf-api` | Secret | Audiobookshelf API credentials |
| CronJob `radigo-{program}` | CronJob | Per-program recording scheduler |

## Reconciliation Contract

### Flux Kustomization Configuration

```yaml
# k8s/flux/apps-kustomization.yaml (existing, add radigo path)
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: apps
  namespace: flux-system
spec:
  interval: 10m
  path: ./k8s/apps
  prune: true
  sourceRef:
    kind: GitRepository
    name: flux-system
  decryption:
    provider: sops
    secretRef:
      name: sops-age
```

### SOPS Decryption

The `secret-audiobookshelf-api.sops.yaml` requires SOPS decryption:

```yaml
# .sops.yaml (existing, ensure pattern matches)
creation_rules:
  - path_regex: k8s/.*\.sops\.yaml$
    age: >-
      <age-public-key>
```

## Health Checks

### Reconciliation Success Criteria

| Check | Command | Expected |
|-------|---------|----------|
| Namespace exists | `kubectl get ns radigo` | Status: Active |
| CronJobs created | `kubectl get cronjobs -n radigo` | Lists all program CronJobs |
| Secret decrypted | `kubectl get secret radigo-audiobookshelf-api -n radigo` | Secret exists |
| ConfigMap created | `kubectl get cm radigo-schedules -n radigo` | ConfigMap exists |

### CronJob Health

| Check | Command | Expected |
|-------|---------|----------|
| Schedule correct | `kubectl get cronjob radigo-arco -n radigo -o jsonpath='{.spec.schedule}'` | Correct cron expression |
| Last job status | `kubectl get jobs -n radigo --sort-by=.status.startTime` | Recent job Completed |
| Pod logs | `kubectl logs -n radigo job/<job-name>` | No errors |

## Failure Modes

### Reconciliation Failures

| Failure | Symptom | Resolution |
|---------|---------|------------|
| SOPS decryption error | Secret not created | Check SOPS age key in flux-system |
| Invalid YAML | Kustomization suspended | Fix syntax, commit, Flux retries |
| Missing dependency | Pod pending | Ensure audiobookshelf PVC exists |

### Runtime Failures

| Failure | Symptom | Resolution |
|---------|---------|------------|
| PVC not mountable | Pod stuck in ContainerCreating | Check node affinity, PVC status |
| Radiko API down | Job fails, logs show curl error | Automatic retry via backoffLimit |
| Audiobookshelf API error | Recording saved but not in library | Manual library scan, check API key |

## Rollback Procedure

1. Revert Git commit with problematic changes
2. Push to main branch
3. Flux automatically reconciles to previous state
4. Verify: `flux get kustomizations -A`

## Monitoring

### Flux Events

```bash
# Watch reconciliation
flux get kustomizations -A --watch

# Check events
kubectl get events -n radigo --sort-by='.lastTimestamp'
```

### CronJob Monitoring

```bash
# List recent jobs
kubectl get jobs -n radigo --sort-by=.status.startTime | tail -10

# Check specific job logs
kubectl logs -n radigo -l job-name=radigo-arco-<timestamp>
```
