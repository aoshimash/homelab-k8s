# Runbook: SQLite Database Cleanup (Post-Migration)

**Feature**: 013-ha-postgres-migration
**Date**: 2026-01-10

## Purpose

Remove the SQLite database file (`home-assistant_v2.db`) from the Home Assistant PVC after the 24-hour observation period has passed and PostgreSQL migration is confirmed stable.

## Prerequisites

- [ ] Migration cutover was executed successfully (PostgreSQL config is active)
- [ ] 24-hour observation period has passed since cutover
- [ ] Home Assistant is running stably with PostgreSQL
- [ ] No rollback is needed (PostgreSQL migration is confirmed successful)
- [ ] Backups are available (Longhorn snapshot and/or CloudNativePG backup)

## Pre-Cleanup Verification

Before deleting SQLite, verify PostgreSQL is working correctly:

```bash
# Verify Home Assistant is using PostgreSQL
kubectl logs -n home-assistant -l app=home-assistant --tail=100 | grep -i "recorder\|postgres"

# Verify PostgreSQL connection is healthy
kubectl run -it --rm psql-check --image=postgres:16 --restart=Never -n postgres -- \
  psql "postgresql://homeassistant:$(kubectl get secret -n home-assistant home-assistant-db-url -o jsonpath='{.data.DB_URL}' | base64 -d | sed 's/.*:\(.*\)@.*/\1/')@postgres-cluster-rw.postgres.svc.cluster.local:5432/homeassistant" \
  -c "SELECT COUNT(*) as total_events FROM events;" \
  -c "SELECT COUNT(*) as total_states FROM states;"

# Verify SQLite file exists (to be deleted)
kubectl run -it --rm check-sqlite --image=busybox --restart=Never -n home-assistant \
  --overrides='{"spec":{"volumes":[{"name":"config","persistentVolumeClaim":{"claimName":"home-assistant-config"}}],"containers":[{"name":"check","image":"busybox","volumeMounts":[{"name":"config","mountPath":"/mnt"}]}]}}' \
  -- ls -lh /mnt/home-assistant_v2.db
```

**Expected**:
- Home Assistant logs show PostgreSQL connection
- PostgreSQL queries return expected row counts
- SQLite file exists in PVC (will be deleted)

## Cleanup Steps

### Step 1: Create Cleanup Job

Apply the cleanup Job manifest:

```bash
# Apply cleanup job
kubectl apply -f k8s/apps/home-assistant/app/job-sqlite-cleanup.yaml

# Watch job progress
kubectl get job -n home-assistant home-assistant-sqlite-cleanup -w

# Check job logs
kubectl logs -n home-assistant -l job-name=home-assistant-sqlite-cleanup -f

# Wait for job completion
kubectl wait --for=condition=complete job/home-assistant-sqlite-cleanup -n home-assistant --timeout=5m
```

**Verification**:
- Job status shows `succeeded: 1`
- Logs show "SQLite database file deleted successfully"
- No errors in job logs

### Step 2: Verify SQLite File Deletion

Confirm the SQLite file has been removed:

```bash
# Verify SQLite file no longer exists
kubectl run -it --rm verify-cleanup --image=busybox --restart=Never -n home-assistant \
  --overrides='{"spec":{"volumes":[{"name":"config","persistentVolumeClaim":{"claimName":"home-assistant-config"}}],"containers":[{"name":"check","image":"busybox","volumeMounts":[{"name":"config","mountPath":"/mnt"}]}]}}' \
  -- ls -lh /mnt/ | grep -i "home-assistant_v2.db" || echo "SQLite file not found (cleanup successful)"
```

**Expected**: SQLite file should not exist (command should output "SQLite file not found")

### Step 3: Verify Home Assistant Still Works

Ensure Home Assistant continues to function normally after cleanup:

```bash
# Check Home Assistant pod status
kubectl get pods -n home-assistant

# Check logs for any errors
kubectl logs -n home-assistant -l app=home-assistant --tail=200 | grep -i error

# Verify PostgreSQL is still being used
kubectl logs -n home-assistant -l app=home-assistant --tail=50 | grep -i "recorder\|postgres"
```

**Expected**:
- Home Assistant pod is Running
- No database-related errors in logs
- Logs confirm PostgreSQL connection

### Step 4: Clean Up Job Manifest

After successful cleanup, remove the cleanup Job from GitOps:

```bash
# Remove job from kustomization.yaml
# (This will be done via Git commit)

# Wait for Flux reconciliation
flux get kustomizations apps -w

# Verify job is removed
kubectl get job -n home-assistant home-assistant-sqlite-cleanup
```

**Expected**: Job should not exist (or be in deletion state)

## Post-Cleanup Checklist

- [ ] SQLite file has been deleted from PVC
- [ ] Home Assistant continues to run normally with PostgreSQL
- [ ] No database-related errors in Home Assistant logs
- [ ] Cleanup Job completed successfully
- [ ] Cleanup Job manifest removed from GitOps
- [ ] Backups are retained per retention policy

## Troubleshooting

### Cleanup Job Fails

**Symptoms**: Job status shows `failed` or `backoffLimitExceeded`

**Resolution**:
1. Check job logs: `kubectl logs -n home-assistant job/home-assistant-sqlite-cleanup`
2. Verify PVC is accessible and writable
3. Verify SQLite file path is correct (`/config/home-assistant_v2.db`)
4. Check for file permissions issues

### Home Assistant Errors After Cleanup

**Symptoms**: Home Assistant shows database errors after SQLite deletion

**Resolution**:
1. Verify Home Assistant is configured to use PostgreSQL (not SQLite)
2. Check `DB_URL` environment variable is set correctly
3. Verify PostgreSQL connectivity
4. Check Recorder configuration in ConfigMap
5. Review Home Assistant logs for specific error messages

### SQLite File Still Exists

**Symptoms**: Verification shows SQLite file still exists after cleanup

**Resolution**:
1. Check cleanup job logs for errors
2. Verify job completed successfully
3. Manually delete if needed (use same cleanup job pattern)
4. Re-run cleanup job if necessary

## Notes

- This cleanup is **irreversible** - ensure backups are available before proceeding
- Keep backups (Longhorn snapshot, CloudNativePG backup) per retention policy
- Only perform cleanup after confirming PostgreSQL migration is stable
- Document cleanup completion date for future reference
- This cleanup should only be performed after the 24-hour observation window
