# Runbook: PostgreSQL Backup Restore and Validation

**Feature**: 013-ha-postgres-migration  
**Date**: 2026-01-09

## Purpose

Validate that Home Assistant database can be restored from CloudNativePG backup, proving disaster recovery capability.

## Prerequisites

- [ ] CloudNativePG cluster `postgres-cluster` is operational
- [ ] At least one successful backup exists (from ScheduledBackup or manual Backup)
- [ ] Test environment available (or ability to create test cluster)
- [ ] Home Assistant can connect to test PostgreSQL cluster

## Restore Procedure

### Step 1: List Available Backups

```bash
# List all backups
kubectl get backup -n postgres

# Get backup details
kubectl describe backup -n postgres <backup-name>

# Check backup destination (R2)
# Backups are stored at: s3://homelab-postgres-backups/cnpg/
```

### Step 2: Create Test PostgreSQL Cluster from Backup

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

### Step 3: Verify Test Cluster Connectivity

```bash
# Get connection credentials
kubectl get secret -n postgres postgres-cluster-test-superuser -o jsonpath='{.data.uri}' | base64 -d

# Test connection
kubectl run -it --rm psql-test --image=postgres:16 --restart=Never -n postgres -- \
  psql "postgresql://postgres:$(kubectl get secret -n postgres postgres-cluster-test-superuser -o jsonpath='{.data.password}' | base64 -d)@postgres-cluster-test-rw.postgres.svc.cluster.local:5432/postgres" \
  -c "SELECT version();"
```

### Step 4: Verify Restored Data Integrity

Connect to the restored database and verify Home Assistant data:

```bash
# Option A: Direct connection to pod (recommended)
POD_NAME=$(kubectl get pods -n postgres -l cnpg.io/cluster=postgres-cluster-test -o jsonpath='{.items[0].metadata.name}')
kubectl exec -it -n postgres $POD_NAME -- psql -U postgres -d homeassistant <<'SQL'

# Option B: Via service (if superuser secret exists)
# kubectl run -it --rm psql-verify --image=postgres:16 --restart=Never -n postgres -- \
#   psql "postgresql://postgres:$(kubectl get secret -n postgres postgres-cluster-test-superuser -o jsonpath='{.data.password}' | base64 -d)@postgres-cluster-test-rw.postgres.svc.cluster.local:5432/homeassistant" <<'SQL'
-- Check table existence
\dt

-- Count records in key tables
SELECT 'events' as table_name, COUNT(*) as row_count FROM events
UNION ALL
SELECT 'states', COUNT(*) FROM states
UNION ALL
SELECT 'events_meta', COUNT(*) FROM events_meta
UNION ALL
SELECT 'states_meta', COUNT(*) FROM states_meta;

-- Check date ranges
SELECT 
  'events' as table_name,
  MIN(created) as earliest,
  MAX(created) as latest,
  COUNT(*) as total
FROM events
UNION ALL
SELECT 
  'states',
  MIN(created) as earliest,
  MAX(created) as latest,
  COUNT(*) as total
FROM states;

-- Check recent entries
SELECT created, event_type, event_data 
FROM events 
ORDER BY created DESC 
LIMIT 10;
SQL
```

**Expected Results**:
- Tables exist (events, states, events_meta, states_meta, etc.)
- Row counts match expectations (should match pre-restore counts)
- Date ranges show historical data
- Recent entries are present

### Step 5: Test Home Assistant Connection to Restored Cluster

Create a temporary Home Assistant configuration pointing to test cluster:

```bash
# Option A: Use app user secret (if superuser secret doesn't exist)
APP_PASSWORD=$(kubectl get secret -n postgres postgres-cluster-test-app -o jsonpath='{.data.password}' | base64 -d)
kubectl create secret generic home-assistant-db-url-test \
  --from-literal=DB_URL="postgresql://app:$APP_PASSWORD@postgres-cluster-test-rw.postgres.svc.cluster.local:5432/homeassistant" \
  -n home-assistant \
  --dry-run=client -o yaml

# Option B: If superuser secret exists
# kubectl create secret generic home-assistant-db-url-test \
#   --from-literal=DB_URL="postgresql://postgres:$(kubectl get secret -n postgres postgres-cluster-test-superuser -o jsonpath='{.data.password}' | base64 -d)@postgres-cluster-test-rw.postgres.svc.cluster.local:5432/homeassistant" \
#   -n home-assistant \
#   --dry-run=client -o yaml

# Note: This is for testing only - do not commit to Git
# Use this Secret temporarily to verify Home Assistant can connect
# Ensure homeassistant database and user exist in test cluster first
```

**Alternative**: Use a test Home Assistant instance or modify existing instance temporarily.

### Step 6: Verify Home Assistant UI Shows Restored Data

If testing with Home Assistant:

1. **Access Home Assistant UI**
2. **Check Historical Charts**:
   - Navigate to History/Lovelace dashboards
   - Verify restored historical data is visible
   - Check date ranges match backup time

3. **Check Logbook**:
   - Open Logbook view
   - Verify entries from backup time are present
   - Check entry counts match expectations

4. **Verify Data Consistency**:
   - Compare row counts between restored DB and original
   - Check for missing or duplicate entries
   - Verify timestamps are correct

## Restore Verification Checklist

- [ ] Test cluster created successfully from backup
- [ ] Test cluster is Ready and accepting connections
- [ ] Database `homeassistant` exists in restored cluster
- [ ] Key tables exist (events, states, etc.)
- [ ] Row counts match expectations
- [ ] Date ranges show historical data
- [ ] Recent entries are present
- [ ] Home Assistant can connect to restored cluster (if tested)
- [ ] Historical data is visible in Home Assistant UI (if tested)

## Cleanup

After validation is complete:

```bash
# Delete test cluster
kubectl delete cluster postgres-cluster-test -n postgres

# Verify cleanup
kubectl get cluster -n postgres
```

## Troubleshooting

### Restore fails

**Symptoms**: Cluster creation fails or stays in "Creating" state

**Resolution**:
1. Check cluster events: `kubectl describe cluster -n postgres postgres-cluster-test`
2. Verify backup exists and is accessible
3. Check CloudNativePG operator logs: `kubectl logs -n cnpg-system deploy/cnpg-controller-manager`
4. Verify R2 credentials are correct (if backup is in R2)
5. Check storage class availability

### Data missing after restore

**Symptoms**: Row counts are lower than expected

**Resolution**:
1. Verify backup was taken after migration completed
2. Check backup status: `kubectl describe backup -n postgres <backup-name>`
3. Verify backup completed successfully (not partial)
4. Check restore logs for errors
5. Consider using a different backup if available

### Home Assistant cannot connect to restored cluster

**Symptoms**: Connection errors when Home Assistant tries to connect

**Resolution**:
1. Verify test cluster service exists: `kubectl get svc -n postgres | grep postgres-cluster-test`
2. Test connectivity from Home Assistant pod network
3. Verify database and user exist in restored cluster
4. Check credentials are correct
5. Verify network policies allow connection

## Notes

- This procedure validates disaster recovery capability
- Test cluster should be deleted after validation
- Document restore time and any issues encountered
- Consider periodic restore testing to ensure backups remain valid
- Restore procedure should complete within reasonable time (< 1 hour for typical homelab scale)
