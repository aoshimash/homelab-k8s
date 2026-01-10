# Runbook: Cutover Execution (SQLite → PostgreSQL)

**Feature**: 013-ha-postgres-migration  
**Date**: 2026-01-09

## Purpose

Execute the cutover sequence to migrate Home Assistant from SQLite to PostgreSQL with planned downtime ≤ 30 minutes.

## Prerequisites

- [ ] Phase 1 & 2 tasks completed (DB/user created, Secret prepared, ConfigMap created)
- [ ] Pre-cutover backups completed (see `runbook-backup.md`)
- [ ] PostgreSQL cluster `postgres-cluster` is Ready
- [ ] Planned downtime window scheduled (≤ 30 minutes)
- [ ] Rollback plan understood and tested

## Cutover Steps

### Step 1: Stop Home Assistant

Scale the Deployment to 0 to prevent writes during migration:

```bash
# Scale down Home Assistant
kubectl scale deployment home-assistant -n home-assistant --replicas=0

# Wait for pod to terminate
kubectl wait --for=delete pod -n home-assistant -l app=home-assistant --timeout=5m

# Verify no pods running
kubectl get pods -n home-assistant
```

**Expected**: No pods should be running in `home-assistant` namespace.

### Step 2: Run Migration Job

Execute the pgloader migration job:

```bash
# The Job manifest should already be applied via GitOps (Flux reconciliation)
# If not, apply manually:
kubectl apply -f k8s/apps/home-assistant/app/job-db-migrate.yaml

# Watch job progress
kubectl get job -n home-assistant home-assistant-db-migrate -w

# Check job logs
kubectl logs -n home-assistant -l job-name=home-assistant-db-migrate -f

# Wait for job completion
kubectl wait --for=condition=complete job/home-assistant-db-migrate -n home-assistant --timeout=30m
```

**Verification**:
- Job status shows `succeeded: 1`
- Logs show "Migration completed successfully!"
- Verification queries in logs show expected row counts

**If job fails**:
- Check logs for errors
- Verify PostgreSQL connectivity and credentials
- Verify SQLite file exists and is readable
- Consider rollback (see `runbook-rollback.md`)

### Step 3: Verify Migration Data

Perform basic data integrity checks:

```bash
# Connect to PostgreSQL and verify data
kubectl run -it --rm psql-check --image=postgres:16 --restart=Never -n postgres -- \
  psql "postgresql://homeassistant:$(kubectl get secret -n home-assistant home-assistant-db-url -o jsonpath='{.data.DB_URL}' | base64 -d | sed 's/.*:\(.*\)@.*/\1/')@postgres-cluster-rw.postgres.svc.cluster.local:5432/homeassistant" \
  -c "SELECT COUNT(*) as total_events FROM events;" \
  -c "SELECT COUNT(*) as total_states FROM states;" \
  -c "SELECT MAX(created) as latest_event FROM events;" \
  -c "SELECT MIN(created) as earliest_event FROM events;"
```

**Expected**: Row counts match expectations, timestamps show historical data range.

### Step 4: Start Home Assistant with PostgreSQL Config

The Deployment should already be configured with PostgreSQL settings (via ConfigMap and Secret). Scale back up:

```bash
# Scale up Home Assistant
kubectl scale deployment home-assistant -n home-assistant --replicas=1

# Watch pod startup
kubectl get pods -n home-assistant -w

# Wait for pod to be Ready
kubectl wait --for=condition=ready pod -n home-assistant -l app=home-assistant --timeout=10m

# Check logs for Recorder initialization
kubectl logs -n home-assistant -l app=home-assistant -f | grep -i recorder
```

**Expected**: 
- Pod starts successfully
- No persistent Recorder DB connection errors
- Logs show successful PostgreSQL connection

### Step 5: Verify History Continuity

Access Home Assistant UI and verify:

1. **Historical Charts**: 
   - Navigate to History/Lovelace dashboards
   - Verify pre-migration data is visible
   - Check date ranges match expectations

2. **Logbook**:
   - Open Logbook view
   - Verify entries from before migration are present
   - Check recent entries are being recorded

3. **New Events**:
   - Trigger a test event (e.g., toggle a switch)
   - Verify new event appears in history/logbook
   - Restart Home Assistant pod
   - Verify new event persists after restart

**Verification Commands**:
```bash
# Check Recorder status via API (if available)
curl -H "Authorization: Bearer $(kubectl get secret -n home-assistant home-assistant-db-url -o jsonpath='{.data.DB_URL}' | base64 -d)" \
  http://home-assistant.home-assistant.svc.cluster.local:8123/api/config

# Check pod logs for errors
kubectl logs -n home-assistant -l app=home-assistant --tail=200 | grep -i error
```

## Post-Cutover Checklist

- [ ] Home Assistant UI loads normally
- [ ] Historical charts show pre-migration data
- [ ] Logbook shows entries from before migration
- [ ] New events are recorded and persist after restart
- [ ] No persistent Recorder DB errors in logs
- [ ] Migration Job completed successfully
- [ ] SQLite database remains accessible in PVC (for 24h rollback window)

## Troubleshooting

### Home Assistant fails to start

**Symptoms**: Pod in CrashLoopBackOff or not Ready

**Resolution**:
1. Check logs: `kubectl logs -n home-assistant -l app=home-assistant`
2. Verify Secret exists: `kubectl get secret -n home-assistant home-assistant-db-url`
3. Verify ConfigMap exists: `kubectl get configmap -n home-assistant home-assistant-recorder-config`
4. Verify PostgreSQL connectivity from pod
5. Check Recorder configuration syntax

### Migration Job fails

**Symptoms**: Job status shows `failed` or `backoffLimitExceeded`

**Resolution**:
1. Check job logs: `kubectl logs -n home-assistant job/home-assistant-db-migrate`
2. Verify SQLite file exists: `kubectl exec -n home-assistant <pod> -- ls -lh /config/home-assistant_v2.db`
3. Verify PostgreSQL credentials and connectivity
4. Check PostgreSQL database/user exists
5. Consider rollback if migration cannot be fixed

### History not visible after migration

**Symptoms**: UI shows no historical data

**Resolution**:
1. Verify migration job completed successfully
2. Check PostgreSQL data: `kubectl exec -it -n postgres <postgres-pod> -- psql -U homeassistant -d homeassistant -c "SELECT COUNT(*) FROM events;"`
3. Verify Home Assistant is connecting to correct database
4. Check Recorder configuration in ConfigMap
5. Review Home Assistant logs for Recorder errors

## Notes

- Total downtime should be ≤ 30 minutes (per spec SC-001)
- Keep SQLite database accessible for ≥ 24 hours for rollback capability
- Document actual downtime duration for future reference
- If cutover exceeds 30 minutes, consider rollback and re-planning
