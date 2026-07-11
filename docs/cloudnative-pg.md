# CloudNativePG - PostgreSQL Cluster Operations

This document describes CloudNativePG PostgreSQL cluster operations, troubleshooting, and management procedures for the homelab-k8s repository.

## Overview

[CloudNativePG](https://cloudnative-pg.io/) is a Kubernetes operator for PostgreSQL, providing automated database cluster management, backups, and high availability.

## Current Configuration

- **PostgreSQL Version**: 16
- **Instances**: 1 (single-node homelab deployment)
- **Storage**: 10GB Longhorn PVC
- **Storage Class**: `longhorn`
- **Backup Schedule**: Daily at 18:00 UTC (03:00 JST)
- **Backup Retention**: 7 days
- **Backup Target**: Cloudflare R2 (S3-compatible)
- **Longhorn Backup Exclusion**: PostgreSQL PVC is excluded from Longhorn recurring backups (backups handled directly by CloudNativePG)

## Configuration Files

### Infrastructure (Operator)

- **Namespace**: `k8s/infrastructure/cloudnative-pg/namespace.yaml`
- **HelmRepository**: `k8s/infrastructure/cloudnative-pg/helmrepository.yaml`
- **HelmRelease**: `k8s/infrastructure/cloudnative-pg/helmrelease.yaml`
- **Kustomization**: `k8s/infrastructure/cloudnative-pg/kustomization.yaml`

### Configs (PostgreSQL Cluster)

- **Namespace**: `k8s/configs/postgres/namespace.yaml`
- **Cluster CRD**: `k8s/configs/postgres/cluster.yaml`
- **ScheduledBackup CRD**: `k8s/configs/postgres/scheduledbackup.yaml`
- **R2 Credentials**: `k8s/configs/postgres/secret-r2-credentials.sops.yaml`
- **Kustomization**: `k8s/configs/postgres/kustomization.yaml`

## Operations

### Check Operator Status

```bash
# Check operator deployment
kubectl get deploy -n cnpg-system

# Check operator logs
kubectl logs -n cnpg-system deploy/cnpg-controller-manager

# Verify CRDs installed
kubectl get crd | grep cnpg
```

### Check PostgreSQL Cluster Status

```bash
# Check cluster status
kubectl get cluster -n postgres

# Check PostgreSQL pods
kubectl get pods -n postgres -l cnpg.io/cluster=postgres-cluster

# Check services
kubectl get svc -n postgres

# Check PVC status
kubectl get pvc -n postgres
```

### Get Connection Credentials

```bash
# Get superuser password
kubectl get secret -n postgres postgres-cluster-superuser -o jsonpath='{.data.password}' | base64 -d

# Get connection string
kubectl get secret -n postgres postgres-cluster-superuser -o jsonpath='{.data.uri}' | base64 -d

# Get app user credentials (if exists)
kubectl get secret -n postgres postgres-cluster-app -o jsonpath='{.data.password}' | base64 -d
```

### Connect to PostgreSQL

#### From Within Cluster

```bash
# Start a psql pod
kubectl run -it --rm psql --image=postgres:16 --restart=Never -- \
  psql "postgresql://postgres:$(kubectl get secret -n postgres postgres-cluster-superuser -o jsonpath='{.data.password}' | base64 -d)@postgres-cluster-rw.postgres.svc.cluster.local:5432/postgres"
```

#### Connection String for Applications

```
postgresql://<user>:<password>@postgres-cluster-rw.postgres.svc.cluster.local:5432/<database>
```

**Service Endpoints**:
- **Read-Write**: `postgres-cluster-rw.postgres.svc.cluster.local:5432`
- **Read-Only**: `postgres-cluster-r.postgres.svc.cluster.local:5432`

## Database Management

### Create Application Database and User

```sql
-- Connect as superuser first
-- Create database for application
CREATE DATABASE myapp;

-- Create user for application
CREATE USER myapp_user WITH ENCRYPTED PASSWORD 'secure_password';

-- Grant privileges
GRANT ALL PRIVILEGES ON DATABASE myapp TO myapp_user;

-- Connect to myapp database and grant schema privileges
\c myapp
GRANT ALL ON SCHEMA public TO myapp_user;
```

### List Databases

```sql
\l
```

### List Users

```sql
\du
```

## Backup Operations

### Check Backup Status

```bash
# List all backups
kubectl get backup -n postgres

# Check scheduled backup status
kubectl get scheduledbackup -n postgres

# Get backup details
kubectl describe backup -n postgres <backup-name>
```

### Create Manual Backup

```bash
# Create on-demand backup
kubectl apply -f - <<EOF
apiVersion: postgresql.cnpg.io/v1
kind: Backup
metadata:
  name: manual-backup-$(date +%Y%m%d-%H%M%S)
  namespace: postgres
spec:
  cluster:
    name: postgres-cluster
EOF

# Check backup status
kubectl get backup -n postgres
```

### Restore from Backup

Restore creates a **new** cluster from an R2 backup via `bootstrap.recovery`, leaving
the production `postgres-cluster` untouched. This doubles as a disaster-recovery drill.
(Verified 2026-06-21: restored the `vikunja` database with row counts matching production.)

```bash
# 1. List available backups (Phase should be "completed")
kubectl get backups.postgresql.cnpg.io -n postgres \
  -o custom-columns=NAME:.metadata.name,PHASE:.status.phase,STOPPED:.status.stoppedAt

# 2. Match the image to production — a mismatched image makes recovery fail
kubectl get cluster -n postgres postgres-cluster -o jsonpath='{.spec.imageName}{"\n"}'

# 3. Create the restore cluster.
#    NOTE: recovery restores the WHOLE cluster (all databases: app/vikunja/paperless/...),
#    not a single database. The referenced Backup object carries its own R2 path + credentials.
kubectl apply -f - <<EOF
apiVersion: postgresql.cnpg.io/v1
kind: Cluster
metadata:
  name: postgres-cluster-test
  namespace: postgres
spec:
  instances: 1
  imageName: ghcr.io/cloudnative-pg/postgresql:16   # match production (step 2)
  storage:
    size: 10Gi
    storageClass: longhorn
  bootstrap:
    recovery:
      backup:
        name: <backup-name>
EOF

# 4. Wait until Ready (CNPG runs a recovery Job first, then starts the pod)
kubectl wait --for=condition=Ready cluster/postgres-cluster-test -n postgres --timeout=30m

# 5. Verify restored data (example: vikunja row counts)
kubectl exec -n postgres postgres-cluster-test-1 -c postgres -- \
  psql -U postgres -d vikunja -c \
  "SELECT 'projects',COUNT(*) FROM projects UNION ALL SELECT 'tasks',COUNT(*) FROM tasks;"

# 6. Cleanup (removes the restore cluster, its pods and PVCs)
kubectl delete cluster -n postgres postgres-cluster-test
```

> **Tip**: Restored row counts match production only if no writes happened after the
> backup was taken. For an exact point-in-time match, pin a `targetTime` via
> `bootstrap.recovery.recoveryTarget` (PITR), which replays WAL up to that timestamp.

## Troubleshooting

### Cluster Not Ready

**Symptoms**: Cluster status shows "Creating" or "Failed"

**Resolution**:
```bash
# Check cluster events
kubectl describe cluster -n postgres postgres-cluster

# Check operator logs
kubectl logs -n cnpg-system deploy/cnpg-controller-manager

# Check PostgreSQL pod logs
kubectl logs -n postgres -l cnpg.io/cluster=postgres-cluster
```

### Pod Not Starting

**Symptoms**: PostgreSQL pod stuck in Pending or CrashLoopBackOff

**Resolution**:
```bash
# Check pod events
kubectl describe pod -n postgres -l cnpg.io/cluster=postgres-cluster

# Check PVC status
kubectl get pvc -n postgres
kubectl describe pvc -n postgres

# Verify Longhorn storage class exists
kubectl get storageclass longhorn
```

### Backup Failed

**Symptoms**: ScheduledBackup shows failed backups

**Resolution**:
```bash
# Check backup details
kubectl describe backup -n postgres <backup-name>

# Verify R2 credentials exist
kubectl get secret -n postgres postgres-r2-credentials

# Check R2 credentials are properly encrypted (SOPS)
# Note: Secret must be encrypted with SOPS before committing to Git
```

### Connection Issues

**Symptoms**: Applications cannot connect to PostgreSQL

**Resolution**:
```bash
# Verify services exist
kubectl get svc -n postgres

# Test DNS resolution from within cluster
kubectl run -it --rm test-dns --image=busybox --restart=Never -- nslookup postgres-cluster-rw.postgres.svc.cluster.local

# Verify cluster is accepting connections
kubectl get cluster -n postgres postgres-cluster -o jsonpath='{.status.conditions[?(@.type=="Ready")].status}'
```

### Storage Issues

**Symptoms**: PVC not bound, pod cannot start

**Resolution**:
```bash
# Check PVC status
kubectl get pvc -n postgres

# Check Longhorn volume status
kubectl get volumes.longhorn.io -n longhorn-system

# Verify Longhorn is operational
kubectl get pods -n longhorn-system
```

### Backup Configuration Notes

**Important**: PostgreSQL PVC is configured to exclude Longhorn recurring backups:
- PVC annotation: `recurring-job-selector.longhorn.io: "[]"`
- This ensures PostgreSQL backups are handled exclusively by CloudNativePG's native backup to R2
- Longhorn volume snapshots are not created for PostgreSQL PVCs
- This prevents duplicate backups and reduces storage costs

**Backup ordering relative to Longhorn PVC backups**: the CNPG daily database
backup (18:00 UTC / 03:00 JST) is deliberately scheduled 30 minutes before
Longhorn's `backup-daily` RecurringJob (18:30 UTC / 03:30 JST). For apps whose
data spans both the CNPG database and a Longhorn PVC (e.g. paperless-ngx,
Immich), a restore must not pair a database backup that is *newer* than the
paired PVC backup: the database could reference files (e.g. media attachments)
missing from the restored volume. The reverse ordering (PVC backup newer than
the database backup) is comparatively safe — at worst a few files exist on disk
that the database doesn't know about yet. Running the database backup
immediately before the PVC backup keeps the two as close as possible while
preserving the safe ordering.

The 30-minute buffer is based on observed CNPG daily backup durations of
**4–25 seconds** (measured 2026-07-03 through 2026-07-10, all backups
completed), so it is a very generous margin. If the database grows enough that
backups approach the buffer, widen the gap and update both schedules together.

## Upgrade Procedures

### Upgrade CloudNativePG Operator

```bash
# Update HelmRelease version in k8s/infrastructure/cloudnative-pg/helmrelease.yaml
# Commit and push changes
# Flux will automatically reconcile the upgrade

# Monitor upgrade progress
kubectl get helmrelease -n cnpg-system cloudnative-pg
kubectl get pods -n cnpg-system
```

### Upgrade PostgreSQL Version

```bash
# Update imageName in k8s/configs/postgres/cluster.yaml
# Example: imageName: ghcr.io/cloudnative-pg/postgresql:17

# Commit and push changes
# CloudNativePG operator will perform rolling upgrade

# Monitor upgrade progress
kubectl get cluster -n postgres postgres-cluster
kubectl get pods -n postgres -l cnpg.io/cluster=postgres-cluster
```

**Note**: PostgreSQL major version upgrades require careful planning. Test in non-production environment first.

## Monitoring

### Prometheus Metrics

CloudNativePG operator and PostgreSQL instances expose Prometheus metrics:

- **Operator**: `cnpg-controller-manager:8080/metrics`
- **PostgreSQL**: `postgres-cluster-1:9187/metrics` (exporter)

Configure Grafana Alloy to scrape these endpoints for monitoring dashboards.

## Security Considerations

### Secret Management

- All secrets are encrypted with SOPS before committing to Git
- R2 credentials stored in `secret-r2-credentials.sops.yaml`
- Superuser credentials auto-generated by CloudNativePG (stored in cluster-managed Secret)

### Network Security

- PostgreSQL only accessible within cluster via ClusterIP services
- No external exposure by default
- Applications connect via internal DNS names

## References

- [CloudNativePG Documentation](https://cloudnative-pg.io/documentation/)
- [CloudNativePG Helm Chart](https://github.com/cloudnative-pg/charts)
- [PostgreSQL Documentation](https://www.postgresql.org/docs/)
