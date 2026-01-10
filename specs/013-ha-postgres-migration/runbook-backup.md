# Runbook: Pre-Cutover Backup Procedures

**Feature**: 013-ha-postgres-migration  
**Date**: 2026-01-09

## Purpose

Document backup procedures to execute before cutover to ensure rollback capability.

## Prerequisites

- Home Assistant PVC `home-assistant-config` exists and is healthy
- CloudNativePG cluster `postgres-cluster` is Ready
- Longhorn is operational
- CloudNativePG backup to R2 is configured

## Backup Procedures

### 1. Longhorn Snapshot (SQLite Source)

Create a snapshot of the Home Assistant PVC to preserve SQLite database state:

```bash
# Option A: Via Longhorn UI
# 1. Navigate to Longhorn UI
# 2. Find volume for PVC home-assistant-config
# 3. Create snapshot with label: pre-migration-$(date +%Y%m%d-%H%M%S)

# Option B: Via kubectl (if Longhorn API available)
# kubectl apply -f - <<EOF
# apiVersion: longhorn.io/v1beta2
# kind: Volume
# metadata:
#   name: <volume-name>
#   namespace: longhorn-system
# spec:
#   # Snapshot creation via Longhorn API
# EOF
```

**Verification**:
```bash
# Check PVC status
kubectl get pvc -n home-assistant home-assistant-config

# Verify Longhorn volume exists
kubectl get volumes.longhorn.io -n longhorn-system | grep home-assistant
```

### 2. CloudNativePG On-Demand Backup (PostgreSQL Baseline)

Create an on-demand backup of the PostgreSQL cluster to establish a baseline:

```bash
# Create on-demand backup
kubectl apply -f - <<EOF
apiVersion: postgresql.cnpg.io/v1
kind: Backup
metadata:
  name: pre-migration-baseline-$(date +%Y%m%d-%H%M%S)
  namespace: postgres
spec:
  cluster:
    name: postgres-cluster
EOF

# Check backup status
kubectl get backup -n postgres

# Wait for backup to complete
kubectl wait --for=condition=completed backup -n postgres -l app=postgres-cluster --timeout=10m
```

**Verification**:
```bash
# List backups
kubectl get backup -n postgres

# Check backup details
kubectl describe backup -n postgres <backup-name>

# Verify backup appears in R2 (if accessible)
# Check CloudNativePG backup destination: s3://homelab-postgres-backups/cnpg/
```

### 3. Manual SQLite File Copy (Optional but Recommended)

For additional safety, copy the SQLite database file outside the PVC:

```bash
# Create a temporary pod to copy the file
kubectl run -it --rm backup-copy --image=busybox --restart=Never -n home-assistant -- \
  sh -c "cp /mnt/config/home-assistant_v2.db /tmp/home-assistant_v2.db.backup && cat /tmp/home-assistant_v2.db.backup" | \
  kubectl exec -i -n home-assistant backup-copy -- cat > /tmp/home-assistant_v2.db.backup

# Or use kubectl cp (if available)
kubectl cp home-assistant/home-assistant-<pod-name>:/config/home-assistant_v2.db \
  /tmp/home-assistant_v2.db.backup
```

**Note**: This step requires pod access and may not be necessary if Longhorn snapshots are reliable.

## Backup Verification Checklist

- [ ] Longhorn snapshot created and listed
- [ ] CloudNativePG backup completed successfully
- [ ] Backup appears in R2 storage (verify via CloudNativePG status)
- [ ] Backup timestamps are recent (within last hour)
- [ ] Document backup names/timestamps for reference

## Rollback Reference

If rollback is needed:
- **SQLite restore**: Use Longhorn snapshot restore or manual file copy
- **PostgreSQL restore**: Use CloudNativePG restore procedure (see `contracts/backup-restore.md`)

## Notes

- Backups should be taken immediately before stopping Home Assistant (during planned downtime window)
- Keep backup artifacts for at least 24 hours after cutover (per spec requirement)
- Document backup names and timestamps in operational log
