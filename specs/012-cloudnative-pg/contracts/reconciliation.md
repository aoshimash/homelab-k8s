# Reconciliation Contract: CloudNativePG PostgreSQL Cluster

**Feature**: 012-cloudnative-pg  
**Date**: 2026-01-07

## Flux Reconciliation Flow

### 1. Infrastructure Layer (k8s/infrastructure/cloudnative-pg/)

```
Git Repository
    │
    ▼
┌───────────────────────────────────────────────────────────────┐
│ Kustomization: infrastructure                                  │
│ Path: ./k8s/infrastructure                                     │
│ Interval: 10m                                                  │
│ Decryption: SOPS                                               │
└───────────────────────────────────────────────────────────────┘
    │
    ▼
┌───────────────────────────────────────────────────────────────┐
│ k8s/infrastructure/cloudnative-pg/kustomization.yaml          │
│                                                                │
│ Resources:                                                     │
│   1. namespace.yaml      → Namespace: cnpg-system              │
│   2. helmrepository.yaml → HelmRepository in flux-system       │
│   3. helmrelease.yaml    → HelmRelease: cloudnative-pg         │
└───────────────────────────────────────────────────────────────┘
    │
    ▼
┌───────────────────────────────────────────────────────────────┐
│ HelmRelease: cloudnative-pg                                    │
│ Namespace: cnpg-system                                         │
│ Chart: cloudnative-pg/cloudnative-pg                           │
│                                                                 │
│ Reconciliation:                                                 │
│   - interval: 10m                                               │
│   - install.remediation.retries: 3                              │
│   - upgrade.remediation.retries: 3                              │
│                                                                 │
│ Installs:                                                       │
│   - CloudNativePG controller manager                            │
│   - CRDs (Cluster, Backup, ScheduledBackup, etc.)              │
│   - RBAC resources                                              │
└───────────────────────────────────────────────────────────────┘
```

### 2. Configs Layer (k8s/configs/postgres/)

```
Git Repository
    │
    ▼
┌───────────────────────────────────────────────────────────────┐
│ Kustomization: configs                                         │
│ Path: ./k8s/configs                                            │
│ Interval: 10m                                                  │
│ Decryption: SOPS                                               │
│ DependsOn: infrastructure                                      │
└───────────────────────────────────────────────────────────────┘
    │
    ▼
┌───────────────────────────────────────────────────────────────┐
│ k8s/configs/postgres/kustomization.yaml                        │
│                                                                │
│ Resources:                                                     │
│   1. namespace.yaml                  → Namespace: postgres     │
│   2. secret-r2-credentials.sops.yaml → Secret (SOPS encrypted) │
│   3. cluster.yaml                    → Cluster CRD             │
│   4. scheduledbackup.yaml            → ScheduledBackup CRD     │
└───────────────────────────────────────────────────────────────┘
    │
    ▼
┌───────────────────────────────────────────────────────────────┐
│ CloudNativePG Operator (watches Cluster CRD)                   │
│                                                                 │
│ Creates:                                                        │
│   - PostgreSQL Pod (StatefulSet-like behavior)                  │
│   - Services (rw, r, ro)                                        │
│   - Secrets (superuser, app credentials)                        │
│   - PVC on Longhorn                                             │
└───────────────────────────────────────────────────────────────┘
```

## Dependency Order

```
1. Namespace (cnpg-system)
   └── 2. HelmRepository (cloudnative-pg)
       └── 3. HelmRelease (cloudnative-pg operator)
           └── 4. CRDs installed
               └── 5. Namespace (postgres)
                   └── 6. Secret (postgres-r2-credentials)
                       └── 7. Cluster (postgres-cluster)
                           └── 8. ScheduledBackup (postgres-cluster-daily)
```

## Health Checks

### Operator Health

| Check | Command | Expected |
|-------|---------|----------|
| Controller Running | `kubectl get deploy -n cnpg-system cnpg-controller-manager` | READY 1/1 |
| CRDs Installed | `kubectl get crd clusters.postgresql.cnpg.io` | Found |

### Cluster Health

| Check | Command | Expected |
|-------|---------|----------|
| Cluster Status | `kubectl get cluster -n postgres postgres-cluster` | STATUS: Cluster in healthy state |
| Pod Running | `kubectl get pods -n postgres -l cnpg.io/cluster=postgres-cluster` | STATUS: Running |
| PVC Bound | `kubectl get pvc -n postgres` | STATUS: Bound |
| Services Created | `kubectl get svc -n postgres` | 3 services (rw, r, ro) |

### Backup Health

| Check | Command | Expected |
|-------|---------|----------|
| ScheduledBackup | `kubectl get scheduledbackup -n postgres` | LAST BACKUP: recent timestamp |
| Backup List | `kubectl get backup -n postgres` | Backups within retention |

## Failure Scenarios

### Scenario 1: Operator Not Ready

**Symptoms**: Cluster CRD stuck in pending, no PostgreSQL pods
**Resolution**: 
```bash
kubectl describe helmrelease -n cnpg-system cloudnative-pg
kubectl logs -n cnpg-system deploy/cnpg-controller-manager
```

### Scenario 2: PVC Not Bound

**Symptoms**: PostgreSQL pod pending, PVC in Pending state
**Resolution**:
```bash
kubectl describe pvc -n postgres
kubectl get storageclass longhorn
```

### Scenario 3: Backup Failed

**Symptoms**: ScheduledBackup shows failed backups
**Resolution**:
```bash
kubectl describe backup -n postgres <backup-name>
# Check R2 credentials
kubectl get secret -n postgres postgres-r2-credentials -o yaml
```

### Scenario 4: Secret Decryption Failed

**Symptoms**: Secret not created, Flux reconciliation error
**Resolution**:
```bash
kubectl get kustomization -n flux-system configs
# Verify SOPS key
kubectl get secret -n flux-system sops-age
```

## Rollback Procedure

### Operator Rollback
```bash
# Revert HelmRelease version in Git
git revert <commit>
git push

# Or manually
flux reconcile helmrelease -n cnpg-system cloudnative-pg
```

### Cluster Rollback
```bash
# Restore from backup
kubectl cnpg backup restore postgres-cluster --backup <backup-name>
```

## Monitoring Endpoints

| Component | Metrics Endpoint | Port |
|-----------|------------------|------|
| Operator | /metrics | 8080 |
| PostgreSQL | /metrics | 9187 (exporter) |

## SLA Targets

| Metric | Target | Measurement |
|--------|--------|-------------|
| Cluster Ready | < 5 min | Time from commit to healthy cluster |
| Backup Completion | < 10 min | Time from scheduled to completed |
| Recovery Time | < 15 min | Time to restore from backup |
