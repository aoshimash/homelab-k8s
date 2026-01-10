# Runbook: Rollback Procedure (PostgreSQL → SQLite)

**Feature**: 013-ha-postgres-migration  
**Date**: 2026-01-09

## Purpose

Restore Home Assistant to SQLite database if migration fails or issues are detected within the 24-hour rollback window.

## Prerequisites

- [ ] Migration cutover was executed (PostgreSQL config is active)
- [ ] Pre-migration SQLite database is accessible in PVC `home-assistant-config`
- [ ] OR Longhorn snapshot is available for restore
- [ ] Decision to rollback has been made (within ≤ 30 minutes of decision per SC-004)

## Rollback Steps

### Step 1: Stop Home Assistant

```bash
# Scale down Home Assistant
kubectl scale deployment home-assistant -n home-assistant --replicas=0

# Wait for pod to terminate
kubectl wait --for=delete pod -n home-assistant -l app=home-assistant --timeout=5m
```

### Step 2: Restore SQLite Database (if needed)

**Option A: SQLite file still exists in PVC** (preferred - no restore needed)

Verify SQLite file exists:
```bash
# Check if SQLite file is still present
kubectl run -it --rm check-sqlite --image=busybox --restart=Never -n home-assistant \
  --overrides='{"spec":{"volumes":[{"name":"config","persistentVolumeClaim":{"claimName":"home-assistant-config"}}],"containers":[{"name":"check","image":"busybox","volumeMounts":[{"name":"config","mountPath":"/mnt"}]}]}}' \
  -- ls -lh /mnt/home-assistant_v2.db
```

**Option B: Restore from Longhorn snapshot**

```bash
# Via Longhorn UI:
# 1. Navigate to Longhorn UI
# 2. Find volume for PVC home-assistant-config
# 3. Select pre-migration snapshot
# 4. Restore snapshot to volume

# Or via kubectl (if Longhorn API available):
# Follow Longhorn restore procedure for the specific snapshot
```

### Step 3: Revert Git Configuration

Revert the commits that switched Home Assistant to PostgreSQL:

```bash
# View recent commits
git log --oneline -10

# Option A: Revert specific commit(s)
git revert <commit-hash>

# Option B: Reset to pre-migration state (if on feature branch)
git reset --hard <pre-migration-commit>

# Push changes (if using GitOps)
git push
```

**Changes to revert**:
- Remove `configmap-recorder.yaml` from kustomization
- Remove `secret-db-url.sops.yaml` from kustomization
- Remove `job-db-migrate.yaml` from kustomization
- Revert Deployment changes (remove DB_URL env var, remove recorder-config volume)

### Step 4: Wait for Flux Reconciliation

```bash
# Watch Flux reconciliation
flux get kustomizations apps -w

# Verify resources are reverted
kubectl get configmap -n home-assistant
kubectl get secret -n home-assistant
kubectl get deployment -n home-assistant -o yaml | grep -A 5 env:
```

### Step 5: Start Home Assistant

```bash
# Scale up Home Assistant (should use SQLite now)
kubectl scale deployment home-assistant -n home-assistant --replicas=1

# Watch pod startup
kubectl get pods -n home-assistant -w

# Wait for pod to be Ready
kubectl wait --for=condition=ready pod -n home-assistant -l app=home-assistant --timeout=10m

# Check logs
kubectl logs -n home-assistant -l app=home-assistant -f
```

### Step 6: Verify Rollback Success

**Verification Checklist**:

- [ ] Home Assistant UI loads normally
- [ ] Historical charts/logbook show data (from SQLite)
- [ ] Core features work:
  - Dashboards display correctly
  - Automations function
  - Device state updates work
- [ ] No user-visible data storage errors in logs
- [ ] Pod runs for 1 hour without data storage issues

**Verification Commands**:
```bash
# Check pod status
kubectl get pods -n home-assistant

# Check logs for errors
kubectl logs -n home-assistant -l app=home-assistant --tail=200 | grep -i error

# Verify SQLite is being used (check logs for Recorder initialization)
kubectl logs -n home-assistant -l app=home-assistant | grep -i "recorder\|sqlite"
```

## Rollback Verification Checklist

After rollback completion, verify for 1 hour:

- [ ] Home Assistant starts successfully
- [ ] UI is accessible and responsive
- [ ] Historical data is visible
- [ ] New events are recorded
- [ ] Automations execute correctly
- [ ] Device state updates work
- [ ] No persistent errors in logs

## Troubleshooting

### Home Assistant fails to start after rollback

**Symptoms**: Pod in CrashLoopBackOff

**Resolution**:
1. Check logs: `kubectl logs -n home-assistant -l app=home-assistant`
2. Verify SQLite file exists and is readable
3. Verify Deployment configuration is reverted (no PostgreSQL references)
4. Check ConfigMap/Secret references are removed
5. Verify PVC is mounted correctly

### Historical data missing after rollback

**Symptoms**: UI shows no history

**Resolution**:
1. Verify SQLite file exists: `kubectl exec -n home-assistant <pod> -- ls -lh /config/home-assistant_v2.db`
2. Check file permissions and ownership
3. Verify SQLite file is not corrupted
4. If file is missing, restore from Longhorn snapshot
5. Check Home Assistant logs for database errors

### Rollback takes longer than 30 minutes

**Symptoms**: Rollback exceeds SC-004 time limit

**Resolution**:
1. Document actual rollback time for post-mortem
2. Identify bottlenecks (snapshot restore, GitOps reconciliation, pod startup)
3. Consider optimizing rollback procedure for future migrations
4. If rollback is incomplete, may need manual intervention

## Notes

- Rollback must complete within ≤ 30 minutes from decision (per SC-004)
- SQLite database should remain accessible for ≥ 24 hours after cutover (per FR-006)
- Document rollback execution time and any issues encountered
- Post-migration data in PostgreSQL will be lost after rollback (explicitly acceptable per spec)
