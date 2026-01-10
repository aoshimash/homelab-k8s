# Runbook: Vikunja Backup Restore and Validation

**Feature**: 014-deploy-vikunja  
**Date**: 2026-01-10

## Purpose

Validate that Vikunja data (database + files) can be restored from backups, proving disaster recovery capability. This runbook covers both database restore (CloudNativePG) and file storage restore (Longhorn).

## Prerequisites

- [ ] CloudNativePG cluster `postgres-cluster` is operational
- [ ] At least one successful backup exists for Vikunja database (from ScheduledBackup or manual Backup)
- [ ] Longhorn volume backups exist for `vikunja-files` PVC
- [ ] Test environment available (or ability to create test resources)
- [ ] Vikunja application can connect to test PostgreSQL cluster

## Restore Procedure

### Part 1: Database Restore (CloudNativePG)

#### Step 1: List Available Backups

```bash
# List all backups
kubectl get backup -n postgres

# Get backup details (look for backups that include vikunja database)
kubectl describe backup -n postgres <backup-name>

# Check backup destination (R2)
# Backups are stored at: s3://homelab-postgres-backups/cnpg/
```

#### Step 2: Create Test PostgreSQL Cluster from Backup

Create a new test cluster restored from backup:

```bash
# Create test cluster from backup
kubectl apply -f - <<EOF
apiVersion: postgresql.cnpg.io/v1
kind: Cluster
metadata:
  name: postgres-cluster-test
  namespace: postgres
spec:
  instances: 1
  imageName: ghcr.io/cloudnative-pg/postgresql:16
  storage:
    size: 10Gi
    storageClass: longhorn
  bootstrap:
    recovery:
      backup:
        name: <backup-name>
        namespace: postgres
EOF

# Wait for cluster to be Ready
kubectl wait --for=condition=Ready cluster/postgres-cluster-test -n postgres --timeout=30m

# Check cluster status
kubectl get cluster -n postgres postgres-cluster-test
```

#### Step 3: Verify Test Cluster Connectivity

```bash
# Get connection credentials
kubectl get secret -n postgres postgres-cluster-test-superuser -o jsonpath='{.data.uri}' | base64 -d

# Test connection
kubectl run -it --rm psql-test --image=postgres:16 --restart=Never -n postgres -- \
  psql "postgresql://postgres:$(kubectl get secret -n postgres postgres-cluster-test-superuser -o jsonpath='{.data.password}' | base64 -d)@postgres-cluster-test-rw.postgres.svc.cluster.local:5432/postgres" \
  -c "SELECT version();"
```

#### Step 4: Verify Restored Database Data Integrity

Connect to the restored database and verify Vikunja data:

```bash
# Option A: Direct connection to pod (recommended)
POD_NAME=$(kubectl get pods -n postgres -l cnpg.io/cluster=postgres-cluster-test -o jsonpath='{.items[0].metadata.name}')
kubectl exec -it -n postgres $POD_NAME -- psql -U postgres -d vikunja <<'SQL'
-- Check table existence
\dt

-- Count records in key tables
SELECT 'users' as table_name, COUNT(*) as row_count FROM users
UNION ALL
SELECT 'projects', COUNT(*) FROM projects
UNION ALL
SELECT 'tasks', COUNT(*) FROM tasks
UNION ALL
SELECT 'task_assignees', COUNT(*) FROM task_assignees
UNION ALL
SELECT 'project_views', COUNT(*) FROM project_views;

-- Check sample data
SELECT id, title, description FROM projects LIMIT 5;
SELECT id, title, done FROM tasks LIMIT 10;

-- Verify user accounts exist
SELECT id, username, email FROM users;
SQL
```

#### Step 5: Update Vikunja to Use Test Database (Optional)

If testing with actual Vikunja application:

```bash
# Update Vikunja DB connection secret to point to test cluster
# (This is for testing only - restore production config after validation)
kubectl patch secret -n vikunja vikunja-db --type='json' -p='[
  {"op": "replace", "path": "/data/VIKUNJA_DATABASE_HOST", "value": "'$(echo -n 'postgres-cluster-test-rw.postgres.svc.cluster.local' | base64)'"}
]'

# Restart Vikunja pods to pick up new connection
kubectl rollout restart deployment -n vikunja vikunja
```

**Verification**:
- Vikunja login page loads
- Existing user can sign in
- Projects and tasks are visible
- Data matches expected state from backup time

### Part 2: Files Restore (Longhorn)

#### Step 1: List Available Longhorn Backups

```bash
# Get PVC volume name
PVC_NAME="vikunja-files"
VOLUME_NAME=$(kubectl get pvc -n vikunja $PVC_NAME -o jsonpath='{.spec.volumeName}')

# List backups via Longhorn UI or API
# Navigate to Longhorn UI → Volumes → Find volume → Backups tab
# Or use Longhorn API if available
```

#### Step 2: Restore Volume from Backup

**Option A: Via Longhorn UI (Recommended)**

1. Navigate to Longhorn UI
2. Find volume for PVC `vikunja-files`
3. Go to Backups tab
4. Select backup to restore
5. Click "Restore" and create new volume or restore to existing

**Option B: Via kubectl (if Longhorn API available)**

```bash
# Create restore operation (example - actual API may vary)
# Refer to Longhorn documentation for exact API format
```

#### Step 3: Update PVC to Use Restored Volume

```bash
# If restored to new volume, update PVC to reference it
# (This requires careful coordination - may need to delete/recreate PVC)
# For testing, consider creating a test PVC that mounts the restored volume
```

#### Step 4: Verify Restored Files

```bash
# Mount restored volume to a test pod
kubectl run -it --rm test-restore --image=busybox --restart=Never -n vikunja --overrides='
{
  "spec": {
    "containers": [{
      "name": "test-restore",
      "image": "busybox",
      "command": ["sh"],
      "stdin": true,
      "tty": true,
      "volumeMounts": [{
        "name": "restored-files",
        "mountPath": "/mnt/files"
      }]
    }],
    "volumes": [{
      "name": "restored-files",
      "persistentVolumeClaim": {
        "claimName": "vikunja-files"
      }
    }]
  }
}' -- sh

# Inside pod, verify files exist
ls -la /mnt/files/
# Check for attachment files, verify timestamps match backup time
```

**Verification**:
- Restored volume is mountable
- Attachment files exist and are accessible
- File timestamps match backup time
- Vikunja can access attachments via UI

### Part 3: Combined Restore Test

For complete validation:

1. **Create test data**:
   - Create test project + tasks in Vikunja
   - Upload test attachment file
   - Note the exact data created

2. **Wait for backups**:
   - Wait for next CNPG ScheduledBackup (or create manual backup)
   - Wait for next Longhorn recurring backup

3. **Perform restore**:
   - Restore database from backup (Part 1)
   - Restore files from backup (Part 2)

4. **Verify restored state**:
   - Sign in to Vikunja
   - Verify test project/tasks exist
   - Verify test attachment is accessible
   - Confirm data matches state at backup time

## Cleanup

After validation:

```bash
# Delete test cluster
kubectl delete cluster -n postgres postgres-cluster-test

# If test PVC was created, delete it
kubectl delete pvc -n vikunja <test-pvc-name>

# Restore Vikunja DB connection to production cluster (if changed)
# kubectl patch secret -n vikunja vikunja-db --type='json' -p='[
#   {"op": "replace", "path": "/data/VIKUNJA_DATABASE_HOST", "value": "'$(echo -n 'postgres-cluster-rw.postgres.svc.cluster.local' | base64)'"}
# ]'
```

## Troubleshooting

### Database Restore Issues

- **Cluster fails to restore**: Check backup status, verify R2 credentials are valid
- **Connection refused**: Verify test cluster is Ready, check service endpoints
- **Missing data**: Verify backup included vikunja database, check backup timestamp

### Files Restore Issues

- **Volume not mountable**: Check Longhorn volume status, verify backup integrity
- **Missing files**: Verify backup timestamp matches expected data, check backup logs
- **Permission errors**: Verify file ownership (Vikunja runs as user 1000)

## Success Criteria

- [ ] Database restore completes successfully
- [ ] Restored database contains expected sample data (projects/tasks)
- [ ] Application can connect to restored database
- [ ] Sign-in works with restored user accounts
- [ ] Files restore completes successfully
- [ ] Restored volume is mountable
- [ ] Sample attachment file is present and accessible
- [ ] Combined restore test validates end-to-end recovery

## References

- CloudNativePG Backup/Restore: https://cloudnative-pg.io/documentation/current/backup_recovery/
- Longhorn Backup/Restore: https://longhorn.io/docs/1.6.0/snapshots-and-backups/
- Vikunja Data Backup: https://vikunja.io/docs/what-to-backup/
