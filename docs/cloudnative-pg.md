# CloudNativePG - PostgreSQL Cluster Operations

This document describes CloudNativePG PostgreSQL cluster operations, troubleshooting, and management procedures for the homelab-k8s repository.

## Overview

[CloudNativePG](https://cloudnative-pg.io/) is a Kubernetes operator for PostgreSQL, providing automated database cluster management, backups, and high availability.

## Current Configuration

- **PostgreSQL Version**: 16
- **Instances**: 1 (single-node homelab deployment)
- **Storage**: one PVC per instance, `10Gi` requested (not enforced; see below)
- **Storage Class**: `local-path` since [#304](https://github.com/aoshimash/homelab-k8s/issues/304); `longhorn` before that. See [Move the cluster to another storage class](#move-the-cluster-to-another-storage-class)
- **Image**: `ghcr.io/tensorchord/cloudnative-vectorchord:16-1.1.1`, with `vchord.so` in `shared_preload_libraries`
- **Backup Schedule**: Daily at 18:00 UTC (03:00 JST)
- **Backup Retention**: 7 days
- **Backup Target**: Cloudflare R2 (S3-compatible), bucket `homelab-postgres-backups`, folder `cnpg/postgres-cluster/` (barman `serverName` defaults to the Cluster name)
- **Volume Backups**: none. PGDATA is marked `k8up.io/backup: "false"` and is not backed up by K8up; CNPG's barman backup to R2 is the only backup. See [Backup Configuration Notes](#backup-configuration-notes)

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

# Check backup status. Use the fully-qualified resource: the short name `backup`
# resolves to `backups.longhorn.io`, which silently reports "No resources found".
kubectl get backups.postgresql.cnpg.io -n postgres
```

An on-demand backup taken this way is subject to the cluster's
`retentionPolicy` (currently `7d`) like any other, so it defines how long the
rollback window for a one-way migration actually lasts.

### Restore from Backup

Restore bootstraps a **new** cluster from the barman backups in R2 via
`bootstrap.recovery`. It never restores in place, and it restores the whole
instance: every database and role, including the `homeassistant` role and
database, which were created by hand and are not declared in this repository.

The procedure below is the restore drill performed on 2026-10-05 for
[#304](https://github.com/aoshimash/homelab-k8s/issues/304), before the cluster
moved to `local-path`. Run it as a drill whenever the restore path needs to be
proven; it leaves the production `postgres-cluster` untouched.

The drill cluster reads the backups through `externalClusters` with the
in-tree `barmanObjectStore` form, the same form the production cluster uses to
write them. It has **no `backup` section**, so it has no backup destination and
cannot write to `homelab-postgres-backups`. Without one, the instance manager
skips WAL archiving entirely (`pkg/management/postgres/archiver/archiver.go` in
CloudNativePG 1.30.1).

```bash
# 1. Confirm a recent completed backup exists. Recovery picks the newest
#    completed backup in the folder and replays archived WAL up to the latest
#    archived segment, unless a recoveryTarget says otherwise.
kubectl get backups.postgresql.cnpg.io -n postgres \
  -o custom-columns=NAME:.metadata.name,PHASE:.status.phase,STOPPED:.status.stoppedAt

# 2. Create the drill cluster. Image and shared_preload_libraries must match
#    production: a physical restore needs the same PostgreSQL major and the same
#    extension binaries.
kubectl apply -f - <<'EOF'
apiVersion: postgresql.cnpg.io/v1
kind: Cluster
metadata:
  name: postgres-restore-drill
  namespace: postgres
spec:
  instances: 1
  inheritedMetadata:
    annotations:
      k8up.io/backup: "false"   # keeps K8upBackupAnnotationMissing quiet
  imageName: ghcr.io/tensorchord/cloudnative-vectorchord:16-1.1.1
  postgresql:
    shared_preload_libraries:
      - "vchord.so"
  storage:
    size: 10Gi
    storageClass: local-path
  # No `backup` section: this cluster must never write to the bucket it
  # restores from.
  bootstrap:
    recovery:
      source: origin
  externalClusters:
    - name: origin
      barmanObjectStore:
        serverName: postgres-cluster   # the folder the production cluster writes to
        destinationPath: "s3://homelab-postgres-backups/cnpg/"
        endpointURL: "https://b2142226728f160a9b43fe01f0fe5f71.r2.cloudflarestorage.com"
        s3Credentials:
          accessKeyId:
            name: postgres-r2-credentials
            key: ACCESS_KEY_ID
          secretAccessKey:
            name: postgres-r2-credentials
            key: ACCESS_SECRET_KEY
        wal:
          maxParallel: 8
EOF

# 3. Wait until Ready. A full-recovery Job runs first, then the instance starts.
#    On 2026-10-05 this took 50 seconds for ~660Mi of PGDATA.
kubectl wait --for=condition=Ready clusters.postgresql.cnpg.io/postgres-restore-drill \
  -n postgres --timeout=30m

# 4. Compare the restored data with production: run the same queries against
#    the drill and the current primary.
PRIMARY=$(kubectl get clusters.postgresql.cnpg.io postgres-cluster -n postgres \
  -o jsonpath='{.status.currentPrimary}')
for POD in postgres-restore-drill-1 "$PRIMARY"; do
  echo "== $POD"
  kubectl exec -n postgres "$POD" -c postgres -- psql -U postgres -d vikunja -Atc \
    "SELECT 'projects',count(*) FROM projects UNION ALL SELECT 'tasks',count(*) FROM tasks
     UNION ALL SELECT 'users',count(*) FROM users"
  kubectl exec -n postgres "$POD" -c postgres -- psql -U postgres -d paperless -Atc \
    "SELECT 'documents',count(*) FROM documents_document"
  kubectl exec -n postgres "$POD" -c postgres -- psql -U postgres -d homeassistant -Atc \
    "SELECT 'states',count(*) FROM states UNION ALL SELECT 'statistics',count(*) FROM statistics
     UNION ALL SELECT 'statistics_meta',count(*) FROM statistics_meta"
  kubectl exec -n postgres "$POD" -c postgres -- psql -U postgres -Atc \
    "SELECT string_agg(datname, ',' ORDER BY datname) FROM pg_database"
  kubectl exec -n postgres "$POD" -c postgres -- psql -U postgres -Atc \
    "SELECT string_agg(rolname, ',' ORDER BY rolname) FROM pg_roles WHERE rolname !~ '^pg_'"
  kubectl exec -n postgres "$POD" -c postgres -- psql -U postgres -Atc \
    "SHOW shared_preload_libraries"
done

# 5. Delete the drill cluster. Its pod and PVC go with it; the PV stays,
#    because the local-path class uses Retain.
kubectl delete clusters.postgresql.cnpg.io -n postgres postgres-restore-drill
```

Then release the drill's PV and remove its directory, following
[local-path-provisioner.md — Release a retained volume](local-path-provisioner.md#release-a-retained-volume).
The directory is `postgres/postgres-restore-drill-1/<pv-name>` under the
provisioner root. Remove the then-empty `postgres-restore-drill-1` directory as
well.

**Result on 2026-10-05.** The drill restored from the R2 folder
`cnpg/postgres-cluster/` onto timeline 2 and reached Ready 50 seconds after
it was created. The same seven databases were present (`app`, `homeassistant`,
`paperless`, `postgres`, `template0`, `template1`, `vikunja`), as were the same
roles, and `shared_preload_libraries` was `vchord.so`. Row counts matched
production for `vikunja` (7 projects, 6 tasks, 1 user), `paperless` (11
documents) and the stable `homeassistant` tables (8 `statistics_meta`, 37052
`statistics`). `homeassistant.states` had 7198 rows against 7201 in production:
the drill's newest row was from 16:03:05 UTC, and production had written three
more by 16:13:05. That gap is the archived-WAL lag: the drill replays only WAL
that is already in R2, and `archive_timeout` here is 5 minutes.

**Restoring for real.** If the production cluster is lost, restore through
Git, not with `kubectl apply`: Flux owns `postgres-cluster` through
`k8s/configs/postgres/cluster.yaml` and would recreate it from that file with a
fresh `initdb`. Change `cluster.yaml` as follows and merge it:

- Keep the name `postgres-cluster`, so the `-rw` Service, the application
  Secrets and the `databases.postgresql.cnpg.io` and
  `scheduledbackups.postgresql.cnpg.io` objects, which all refer to that name,
  keep working. Keep `backup`, `managed.roles`, the image and the preload.
- Add `bootstrap.recovery` and `externalClusters` exactly as in the drill
  manifest above.
- Set `backup.barmanObjectStore.serverName` to a name that is not
  `postgres-cluster` (for example `postgres-cluster-v2`). CloudNativePG's
  recovery documentation says not to share one object store configuration
  between backup and recovery unless each cluster has its own `serverName`, so
  that archiving cannot overwrite the backups being restored. If the folder is
  not empty, its safety check stops the cluster in `Setting up primary` with
  `Expected empty archive` in the pod log.

`bootstrap` is only read when the Cluster object is created. If a broken
`postgres-cluster` object still exists, stop Flux from touching it before the
change merges: `flux suspend kustomization configs`, delete the object, merge,
then `flux resume kustomization configs`, so Flux creates it from the recovery
spec. Merging while the old instance still runs would point its archiving at
the new `serverName` folder, and the recovered cluster would then stop on
`Expected empty archive`. Remove `bootstrap` and
`externalClusters` again in a follow-up change once the cluster is running.

> **Tip**: For an exact point-in-time match, set
> `bootstrap.recovery.recoveryTarget.targetTime`, which replays WAL only up to
> that timestamp.

> **Deprecation**: the in-tree `barmanObjectStore` form used here and in
> `cluster.yaml` is deprecated in favour of the Barman Cloud Plugin, and the
> CloudNativePG 1.30 release notes say it will be removed in 1.31.0. Both backup
> and this restore procedure have to move to the plugin before the operator is
> upgraded to 1.31.

### Move the cluster to another storage class

`spec.storage.storageClass` can be changed on an existing Cluster. In
CloudNativePG 1.30.1 the validating webhook rejects only a smaller
`storage.size` (`validateStorageConfigurationChange` in
`internal/webhook/v1/cluster_webhook.go`), and the PVC reconciler adjusts only
the size and VolumeAttributesClass of existing PVCs
(`pkg/reconciler/persistentvolumeclaim/existing.go`). The new class therefore
applies only to PVCs created after the change, and moving the data means
replacing the instance. CloudNativePG's storage documentation describes the
same idea for a multi-instance cluster: re-create each instance on a new PVC.
This cluster has one instance, so it gets a temporary second one instead. That
keeps the Cluster name, its Services and Secrets, the application connection
strings, and the WAL archive folder. It is how the cluster moved from
`longhorn` to `local-path` in [#304](https://github.com/aoshimash/homelab-k8s/issues/304).

Do not run it between 17:45 and 19:30 UTC, when the CNPG daily backup (18:00)
and the volume backups run.

1. **Prove the backups restore, and take a fresh one.** Run the drill in
   [Restore from Backup](#restore-from-backup), then take an on-demand backup
   ([Create Manual Backup](#create-manual-backup)) and wait for `completed`.
2. **Keep the old volume.** Set the current instance's PV to `Retain`, so that
   removing the old instance later leaves its data behind:

   ```bash
   # the instance being replaced: the current (only) primary
   OLD=$(kubectl get clusters.postgresql.cnpg.io postgres-cluster -n postgres \
     -o jsonpath='{.status.currentPrimary}')
   PV=$(kubectl get pvc -n postgres "$OLD" -o jsonpath='{.spec.volumeName}')
   kubectl patch pv "$PV" -p '{"spec":{"persistentVolumeReclaimPolicy":"Retain"}}'
   ```

   For a move to `local-path`, also run
   [Check the user volume before creating a claim](local-path-provisioner.md#check-the-user-volume-before-creating-a-claim).
3. **Add an instance on the new class.** In `cluster.yaml`, set the new
   `storageClass` and `instances: 2`, and merge. The operator clones a new
   replica from the primary onto a PVC of the new class. Wait until the
   Cluster reports both instances ready, the replica streams, and its replayed
   LSN has reached the primary's current one (`replay_lag` alone is not
   enough: it is empty when the primary is idle):

   ```bash
   kubectl get clusters.postgresql.cnpg.io,pods,pvc -n postgres
   PRIMARY=$(kubectl get clusters.postgresql.cnpg.io postgres-cluster -n postgres \
     -o jsonpath='{.status.currentPrimary}')
   kubectl exec -n postgres "$PRIMARY" -c postgres -- psql -U postgres -Atc \
     "SELECT application_name, state, pg_current_wal_lsn(), replay_lsn FROM pg_stat_replication"
   ```

4. **Switch over to the new instance.** With the `kubectl cnpg` plugin:
   `kubectl cnpg promote postgres-cluster <new-instance>`. Without it, set the
   status fields the plugin sets (`internal/cmd/plugin/promote/promote.go` in
   1.30.1). The plugin also checks that the target pod exists and is not
   fenced, and sets the Cluster's `Ready` condition to `False`. The patch below
   does neither, so check the pod by hand first:

   ```bash
   # The instance on the new storage class. The operator reuses the lowest
   # free serial, so take it from the PVC on the new class (an instance is
   # named after its PVC) rather than assuming -2.
   NEW=$(kubectl get pvc -n postgres -l cnpg.io/cluster=postgres-cluster \
     -o jsonpath='{.items[?(@.spec.storageClassName=="local-path")].metadata.name}')
   kubectl get pod -n postgres "$NEW"
   kubectl patch clusters.postgresql.cnpg.io postgres-cluster -n postgres \
     --subresource=status --type=merge -p "{\"status\":{
       \"targetPrimary\":\"$NEW\",
       \"targetPrimaryTimestamp\":\"$(date -u +%Y-%m-%dT%H:%M:%S.000000Z)\",
       \"phase\":\"Switchover in progress\",
       \"phaseReason\":\"Switching over to $NEW\"}}"
   ```

   The former primary checkpoints, shuts down fast and comes back as a replica
   of the new one. Then confirm that writes work, that the applications
   reconnected, that WAL archiving continues
   (`ContinuousArchiving` is `True` and `pg_stat_archiver` advances on the new
   primary), and that an on-demand backup completes.

   One failed archive attempt right after the promotion is expected. While the
   Cluster status still names the old primary, the new primary's `wal-archive`
   logs `switchover in progress, refusing archiving`, and PostgreSQL retries a
   second later. `pg_stat_archiver.failed_count` keeps that 1.

   The cluster's backup `target` is the default, `prefer-standby`, so while the
   old instance is still a replica an on-demand backup runs on it, not on the
   new primary. Take another backup after step 5, when the new instance is the
   only one.
5. **Remove the old instance.** First check that both pods are running and
   ready, that the new instance is the primary, and that the old one is a
   streaming replica. On scale-down the operator first removes an instance that
   has no pod, with no primary check. Otherwise it skips the primary and removes
   the ready replica with the highest serial (`findDeletableInstance` in
   `internal/controller/replicas.go`). With both pods up, that is the old
   instance:

   ```bash
   kubectl get pods -n postgres -L cnpg.io/instanceRole
   ```

   Then set `instances: 1` and merge. The operator deletes the old instance and
   its PVC. Its PV stays `Released` because of step 2.

**What happened in #304 (2026-10-05, UTC).**

- Steps 1–2: the drill and the backup `pre-local-path-304-20261005` ran at
  16:13–16:16, and the Longhorn PV `pvc-a37d3284-abfb-4b6e-9887-96255f0385b6`
  (instance `postgres-cluster-1`) was set to `Retain`.
- Step 3: Flux applied the change at 16:33. `postgres-cluster-2` was cloned
  onto `local-path` and was a streaming replica at the primary's LSN within
  about a minute.
- Step 4: the status patch at 16:34:26 promoted `postgres-cluster-2` on
  timeline 2 about 5 seconds later. A `pg_isready` loop against
  `postgres-cluster-rw` failed from 16:34:26 to 16:34:34 and succeeded again at
  16:34:35, so the write endpoint was down for about 9 seconds.
  `postgres-cluster-1` restarted once, ran `pg_rewind` (which reported
  `no rewind required`) and was streaming from the new primary at 16:34:38. The
  Cluster reported both instances ready at 16:34:48.
- Applications: Home Assistant's recorder logged one `SSL connection has been
  closed unexpectedly` error and reconnected on its own. Vikunja and
  Paperless-ngx needed no restart: afterwards Vikunja's `/health`, which pings
  the database, returned `OK`, and Paperless-ngx's ORM counted its 11 documents.
- Archiving continued into the same `cnpg/postgres-cluster/` folder:
  `00000002.history`, then `000000010000011300000013.partial` (the old
  timeline's last segment), then `000000020000011300000013` on the new
  timeline. The on-demand backup `post-switchover-304-20261005` completed,
  taken on `postgres-cluster-1` because of `prefer-standby`.
- Step 5 removed `postgres-cluster-1` and its PVC. Its Longhorn PV,
  `pvc-a37d3284-abfb-4b6e-9887-96255f0385b6`, was deliberately left `Released`
  (reclaim policy `Retain`) as the last copy of the pre-move PGDATA. A
  `Retain` PV is never deleted automatically: delete it, and its Longhorn
  volume, when Longhorn is removed
  ([#305](https://github.com/aoshimash/homelab-k8s/issues/305)).

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

# Verify the storage class exists
kubectl get storageclass local-path
```

A `local-path` claim stays `Pending` until its pod is scheduled
(`WaitForFirstConsumer`), and stays `Pending` for good if the node's user
volume is missing. See
[local-path-provisioner.md — Check the user volume before creating a claim](local-path-provisioner.md#check-the-user-volume-before-creating-a-claim).

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

# Which directory on the node each instance's volume is
kubectl get pv -o custom-columns=NAME:.metadata.name,CLAIM:.spec.claimRef.name,CLASS:.spec.storageClassName,PATH:.spec.hostPath.path \
  | awk 'NR==1 || $2 ~ /^postgres-cluster-/'

# Provisioner and its helper pods
kubectl get pods -n local-path-storage
kubectl get events -n local-path-storage --field-selector type=Warning
```

`local-path` does not enforce the requested size. PGDATA can grow until the
node's `EPHEMERAL` filesystem is full; see
[local-path-provisioner.md — Capacity](local-path-provisioner.md#capacity).

### Backup Configuration Notes

**PGDATA has no volume-level backup.** The instance PVCs carry
`k8up.io/backup: "false"` through the Cluster's `spec.inheritedMetadata`, and the
`postgres` namespace has no K8up Schedule, so K8up never copies them. A
file-level copy of a running PostgreSQL data directory is not a valid backup.
CNPG's barman backup to R2 is the only backup of the databases, and the only
source the [Restore from Backup](#restore-from-backup) procedure uses.

While PGDATA was on Longhorn (until [#304](https://github.com/aoshimash/homelab-k8s/issues/304)),
it was also copied by Longhorn's `backup-daily` RecurringJob, because Longhorn
puts every volume in its `default` recurring-job group unless the StorageClass
says otherwise. Those copies were crash-consistent snapshots of a running
PGDATA, never the restore path.

**Backup ordering relative to Longhorn PVC backups**: the CNPG daily database
backup (18:00 UTC / 03:00 JST) is deliberately scheduled 30 minutes before
Longhorn's `backup-daily` RecurringJob (18:30 UTC / 03:30 JST). For apps whose
data spans both the CNPG database and a Longhorn PVC (e.g. paperless-ngx), a
restore must not pair a database backup that is *newer* than the
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
- **PostgreSQL**: `<instance>:9187/metrics` on each instance pod, e.g. `postgres-cluster-2` (exporter)

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
