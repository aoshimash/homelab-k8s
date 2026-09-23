# K8up - Volume Backups to R2

K8up backs up selected PersistentVolumeClaims to Cloudflare R2 with restic. It
replaces Longhorn's recurring backups as part of the storage migration recorded
in [storage-migration-decision.md](storage-migration-decision.md); implemented
by [#301](https://github.com/aoshimash/homelab-k8s/issues/301).

Backups are **opt-in**: a volume is backed up only when its PVC is explicitly
marked. PostgreSQL is not backed up here — CloudNativePG ships its own barman
backups to R2 (see [cloudnative-pg.md](cloudnative-pg.md)).

## Current Configuration

| Item | Value |
|------|-------|
| Chart | `k8up/k8up`, version pinned in `k8s/infrastructure/k8up/helmrelease.yaml`; CRDs bundled in the chart |
| Operator namespace | `k8up` |
| Selection | `k8up.skipWithoutAnnotation: true` — only PVCs annotated `k8up.io/backup: "true"` |
| Target | R2 bucket `homelab-k8up-backups`, one restic repository per namespace at `homelab-k8up-backups/<namespace>` |
| Backup | daily, 19:00 UTC |
| Prune | Sundays 20:00 UTC, `keepDaily: 30` |
| Check | Sundays 21:00 UTC (`restic check`) |
| Job user | root (`podSecurityContext.runAsUser: 0`) |

### Backed-up volumes

| Namespace | PVC | Notes |
|-----------|-----|-------|
| audiobookshelf | `audiobookshelf-config` | `absdatabase.sqlite*` excluded from the file copy; the database is captured by a command-based dump instead (below) |
| audiobookshelf | `audiobookshelf-metadata` | |
| audiobookshelf | `audiobookshelf-podcasts` | Most feeds have ended, so re-fetching is not possible |
| home-assistant | `home-assistant-config` | |
| paperless-ngx | `paperless-data` | |
| paperless-ngx | `paperless-media` | Scanned originals |
| vikunja | `vikunja-files` | Task attachments |

Not backed up: `postgres/postgres-cluster-1` (CloudNativePG PGDATA — a
file-level copy of a live PostgreSQL data directory is not a valid backup). It
carries no annotation and the `postgres` namespace has no Schedule, so it is
excluded twice over.

### Why these settings

- **`skipWithoutAnnotation: true`.** The chart default is `false`, under which
  every PVC in a namespace that has a Schedule is backed up. That default is
  exactly the failure mode described in the decision record: under Longhorn,
  PGDATA was backed up for months because nothing named it for exclusion.
- **One repository per namespace.** A restic `prune` takes an exclusive lock on
  its repository. Separate repositories keep one namespace's weekly prune from
  blocking another namespace's backup, and let a namespace be restored or
  deleted without touching the others.
- **`keepDaily: 30`.** The decision record left retention to be set here rather
  than carried over. 30 daily snapshots keeps the recovery window Longhorn's
  `backup-daily` RecurringJob had when K8up was introduced (`retain: 30` as of
  `5dc5e70`), so moving to K8up does not shorten how far back a volume can be
  recovered. restic deduplicates, so extra daily snapshots of mostly-unchanged
  data cost little in R2.
- **19:00 UTC.** Longhorn's `backup-daily` started at 18:30 UTC
  (`cron: "30 18 * * *"` as of `5dc5e70`) and keeps running while both systems
  coexist; starting K8up half an hour later avoids both reading the same
  volumes from the same moment.
- **Root job pods.** The applications run as different users (audiobookshelf
  99, paperless-ngx and vikunja 1000, Home Assistant root). Running backups as
  root lets restic read every file whatever its owner and mode, and running
  restores as root is what lets restic write files back with their original
  ownership. The cluster's PodSecurity enforce level is baseline, which permits
  it.
- **Per-namespace Secrets, not operator-global credentials.** K8up can take
  global credentials from operator environment variables
  (`BACKUP_GLOBALACCESSKEYID` and friends), but the operator then writes their
  values into each backup Job's pod spec as plain `env` values. Per-namespace
  Secrets are referenced by `secretKeyRef` instead, so the credentials stay in
  Secrets.

## Configuration Files

```
k8s/infrastructure/k8up/
├── namespace.yaml
├── helmrepository.yaml       # https://k8up-io.github.io/k8up
├── helmrelease.yaml          # skipWithoutAnnotation, timezone, CRD policy
└── kustomization.yaml

k8s/apps/<app>/app/
├── pvc.yaml                  # k8up.io/backup: "true" on backed-up PVCs
├── k8up-schedule.yaml        # Schedule "backup": backend, schedule, retention
└── secret-k8up-backup.sops.yaml  # Secret "k8up-backup" (SOPS)
```

`k8s/flux/configs-kustomization.yaml` health-checks the `k8up` HelmRelease, so
the Schedule CRD exists before the `apps` layer (which depends on `configs`)
applies the per-app Schedules.

### Credentials

Each namespace with a Schedule has a Secret `k8up-backup` with three keys:

| Key | Content |
|-----|---------|
| `ACCESS_KEY_ID` | R2 API token access key, scoped to `homelab-k8up-backups` (Object Read & Write) |
| `SECRET_ACCESS_KEY` | its secret |
| `RESTIC_PASSWORD` | restic repository password, identical in every namespace |

**The restic password is the only key to the backup data.** Without it the
repositories in R2 cannot be read, and restic cannot recover a lost password.
It is kept in Git, SOPS-encrypted, so the recovery chain is: the age key
(`age.agekey`, kept outside Git) → any `secret-k8up-backup.sops.yaml` →
`RESTIC_PASSWORD`. Losing the age key loses the backups along with every other
secret in this repository.

Read a value when needed (e.g. for a manual restic session):

```bash
SOPS_AGE_KEY_FILE=age.agekey sops -d k8s/apps/vikunja/app/secret-k8up-backup.sops.yaml
```

## audiobookshelf: command-based SQLite dump

audiobookshelf keeps its state in `/config/absdatabase.sqlite`. restic reads the
live volume, and a file-level copy of an open SQLite database is not
crash-consistent. Two annotations handle it:

1. The Deployment's pod template carries `k8up.io/backupcommand`. At backup
   time K8up runs it inside the running container and stores its stdout in
   the namespace's repository as a separate snapshot, file
   `/audiobookshelf-audiobookshelf.sqlite` (`/<namespace>-<container>` plus the
   `k8up.io/file-extension`). The command uses SQLite's `VACUUM INTO` to write a
   transactionally consistent copy to `/tmp`, streams it, and deletes it. The
   image has no `sqlite3` CLI, so it drives the app's own node `sqlite3` module.
   Any failure exits non-zero, which fails the backup job rather than storing a
   partial dump.
2. The `audiobookshelf-config` PVC carries
   `k8up.io/backup-restic-args: '["--exclude", "absdatabase.sqlite*"]'`, so the
   file-level backup of the volume does not also store a torn copy of the
   database. Everything else on the volume (`migrations/` etc.) is copied as
   files.

**If the audiobookshelf pod is not Running at backup time, that day gets no
database snapshot — and nothing fails.** K8up only runs backup commands in
Running pods, and when it finds none it creates no dump job at all, so the
Backup still reports success. Because the file copy excludes the live database,
the newest database snapshot is then simply older than the rest. Older dump
snapshots are kept by retention, so what is lost is freshness, not the
database. Check the date of the newest `/audiobookshelf-audiobookshelf.sqlite`
snapshot (see "List snapshots") after any day audiobookshelf was down at
19:00 UTC.

Adding a new application with an embedded database: use the same pattern — a
backup command that streams a consistent dump, plus an exclude for the live
file.

## Adding a Volume to Backups

1. Annotate the PVC:

   ```yaml
   metadata:
     annotations:
       k8up.io/backup: "true"
   ```

2. If the namespace has no `k8up-schedule.yaml` yet, copy one from another app
   (change `metadata.namespace` and the `bucket` prefix), and add a
   `secret-k8up-backup.sops.yaml` with the same three keys — copy an existing
   one, `sops -d` it, change the namespace, and re-encrypt. List both in the
   app's `kustomization.yaml`.
3. After it reconciles, trigger a backup (below) and confirm a snapshot for
   `/data/<pvc>` appears.

To exclude a volume deliberately, leave it unannotated or set
`k8up.io/backup: "false"`.

## Operations

Use the fully qualified resource names. Short names are ambiguous in this
cluster: `backup` also matches Longhorn's `backups.longhorn.io` and
CloudNativePG's `backups.postgresql.cnpg.io`.

### Check status

```bash
# Operator and CRDs
kubectl -n k8up get pods
flux get helmreleases -n k8up
kubectl get crd | grep k8up.io

# Schedules and the jobs they created
kubectl get schedules.k8up.io -A
kubectl get backups.k8up.io,prunes.k8up.io,checks.k8up.io -A
kubectl -n <namespace> get jobs

# Why a PVC was or was not included (operator logs list each PVC)
kubectl -n k8up logs deploy/k8up | grep -i pvc
```

### Trigger an on-demand backup

The Backup takes its backend and security context from the namespace's
Schedule, so the two cannot drift:

```bash
NS=vikunja
kubectl -n "$NS" get schedules.k8up.io backup -o json \
  | jq '{apiVersion: "k8up.io/v1", kind: "Backup",
         metadata: {generateName: "manual-"},
         spec: {backend: .spec.backend, podSecurityContext: .spec.podSecurityContext}}' \
  | kubectl -n "$NS" create -f -

kubectl -n "$NS" get backups.k8up.io -w
```

### List snapshots

K8up mirrors the repository's snapshots as `Snapshot` resources. The path tells
which volume a snapshot holds: `/data/<pvc>` for a volume, or
`/<namespace>-<container><ext>` for a command-based dump.

```bash
kubectl -n "$NS" get snapshots.k8up.io \
  -o custom-columns='ID:.spec.id,DATE:.spec.date,PATHS:.spec.paths'
```

## Restore

Restore into a **new** PVC first and verify it, rather than writing over a
live volume. Put restored data into service only once it checks out.

### Restore a volume into a new PVC

```bash
NS=vikunja
PVC=vikunja-files
SNAPSHOT=<id from "List snapshots">   # the /data/$PVC snapshot to restore
SC=$(kubectl -n "$NS" get pvc "$PVC" -o jsonpath='{.spec.storageClassName}')

# 1. Target PVC. Annotated "false" so it is never picked up by a backup.
kubectl -n "$NS" apply -f - <<EOF
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: restore-$PVC
  annotations:
    k8up.io/backup: "false"
spec:
  accessModes: [ReadWriteOnce]
  storageClassName: $SC
  resources:
    requests:
      storage: 10Gi   # at least the volume's actual size
EOF

# 2. Restore job, backend copied from the Schedule. `paths` makes the restore
#    fail if the snapshot is not the one for this PVC.
kubectl -n "$NS" get schedules.k8up.io backup -o json \
  | jq --arg snap "$SNAPSHOT" --arg pvc "$PVC" \
      '{apiVersion: "k8up.io/v1", kind: "Restore",
        metadata: {name: ("restore-" + $pvc)},
        spec: {snapshot: $snap, paths: [("/data/" + $pvc)],
               restoreMethod: {folder: {claimName: ("restore-" + $pvc)}},
               backend: .spec.backend,
               podSecurityContext: .spec.podSecurityContext}}' \
  | kubectl -n "$NS" create -f -

# 3. Wait for the job to complete (kubectl can watch only one resource type)
kubectl -n "$NS" get restores.k8up.io -w
```

The snapshot's `/data/<pvc>` prefix is stripped on restore, so the files land
at the root of the new PVC, as they were on the original.

**A `Completed` Restore does not prove the restore worked.** In K8up v2.16.0 a
folder restore runs restic and returns success regardless of restic's exit
status, so a failed or partial restore still completes. The checksum
comparison below is the actual check — never skip it.

### Verify

Mount the live volume read-only next to the restored one and compare
checksums. Files changed since the snapshot was taken will differ; anything
else should not.

```bash
kubectl -n "$NS" apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: restore-verify
spec:
  restartPolicy: Never
  securityContext:
    runAsUser: 0
  containers:
    - name: verify
      image: alpine:3.22
      command: ["sleep", "3600"]
      volumeMounts:
        - {name: live, mountPath: /live, readOnly: true}
        - {name: restored, mountPath: /restored, readOnly: true}
  volumes:
    - {name: live, persistentVolumeClaim: {claimName: $PVC, readOnly: true}}
    - {name: restored, persistentVolumeClaim: {claimName: restore-$PVC}}
EOF
kubectl -n "$NS" wait --for=condition=Ready pod/restore-verify

kubectl -n "$NS" exec restore-verify -- sh -c '
  cd /live     && find . -type f ! -path "./lost+found/*" -exec sha256sum {} + | sort -k 2 > /tmp/live.sum
  cd /restored && find . -type f ! -path "./lost+found/*" -exec sha256sum {} + | sort -k 2 > /tmp/restored.sum
  wc -l /tmp/live.sum /tmp/restored.sum
  diff /tmp/live.sum /tmp/restored.sum && echo IDENTICAL'
```

The pod must run on the node the live volume is attached to, which is
automatic on a single node.

For `audiobookshelf-config`, `absdatabase.sqlite*` shows up as missing from the
restored copy. That is expected: the file copy excludes the live database, which
lives in its own snapshot (see "Restore audiobookshelf's database").

### Put restored data into service

Once the copy in `restore-<pvc>` checks out, restore the same snapshot onto
the **original** PVC — the one defined in Git, which keeps its
`k8up.io/backup: "true"` annotation. Do not repoint the Deployment at
`restore-<pvc>`: that PVC exists only in the cluster and is annotated `"false"`,
so the application would run on a volume that is neither in Git nor backed up.

```bash
APP=vikunja   # the Deployment using $PVC

# 1. Stop Flux from reverting the scale-down (it re-applies replicas: 1 from
#    Git on every reconcile), then stop the application.
flux suspend kustomization apps
kubectl -n "$NS" scale deploy/"$APP" --replicas=0

# 2. Restore onto the original PVC. `delete: true` removes files that are not
#    in the snapshot, so the volume ends up exactly as it was backed up.
#    For audiobookshelf-config that includes the live database, which the file
#    snapshot excludes: do "Restore audiobookshelf's database" afterwards,
#    before step 4.
kubectl -n "$NS" get schedules.k8up.io backup -o json \
  | jq --arg snap "$SNAPSHOT" --arg pvc "$PVC" \
      '{apiVersion: "k8up.io/v1", kind: "Restore",
        metadata: {name: ("restore-inplace-" + $pvc)},
        spec: {snapshot: $snap, paths: [("/data/" + $pvc)], delete: true,
               restoreMethod: {folder: {claimName: $pvc}},
               backend: .spec.backend,
               podSecurityContext: .spec.podSecurityContext}}' \
  | kubectl -n "$NS" create -f -
kubectl -n "$NS" get restores.k8up.io -w

# 3. Verify the original PVC now matches the copy verified above — Completed
#    is not proof (see above). restore-verify still mounts both volumes.
kubectl -n "$NS" exec restore-verify -- sh -c '
  cd /live     && find . -type f ! -path "./lost+found/*" -exec sha256sum {} + | sort -k 2 > /tmp/live.sum
  cd /restored && find . -type f ! -path "./lost+found/*" -exec sha256sum {} + | sort -k 2 > /tmp/restored.sum
  diff /tmp/live.sum /tmp/restored.sum && echo IDENTICAL'

# 4. Only when IDENTICAL: hand control back to Flux; resuming re-applies
#    replicas: 1.
flux resume kustomization apps
```

If the original PVC is gone entirely, recreate it from its manifest in Git
(`kubectl apply -f k8s/apps/<app>/app/pvc.yaml` — identical to what Flux would
apply) while `apps` is still suspended and the application is scaled to zero,
then restore into it as in step 2. Resuming `apps` before the restore would
start the application on an empty volume.

### Restore audiobookshelf's database

The database lives in its own snapshot (`/audiobookshelf-audiobookshelf.sqlite`)
and is not part of the `audiobookshelf-config` snapshot. Extract it with
`restic dump`, using the restic binary in the K8up image:

```bash
NS=audiobookshelf
SNAPSHOT=<id of the /audiobookshelf-audiobookshelf.sqlite snapshot>
REPO=$(kubectl -n "$NS" get schedules.k8up.io backup -o json \
  | jq -r '.spec.backend.s3 | "s3:\(.endpoint)/\(.bucket)"')
# restic comes from the image the running operator uses
IMG=$(kubectl -n k8up get deploy k8up -o jsonpath='{.spec.template.spec.containers[0].image}')

# Stop audiobookshelf so nothing holds the database open (suspend first, or
# Flux scales it back up)
flux suspend kustomization apps
kubectl -n "$NS" scale deploy/audiobookshelf --replicas=0

kubectl -n "$NS" apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: restore-absdb
spec:
  restartPolicy: Never
  securityContext:
    runAsUser: 0
  containers:
    - name: restic
      image: $IMG
      command: ["sh", "-c"]
      args:
        - restic dump "$SNAPSHOT" /audiobookshelf-audiobookshelf.sqlite > /config/absdatabase.sqlite.restored
          && chown 99:99 /config/absdatabase.sqlite.restored
      env:
        - {name: RESTIC_REPOSITORY, value: "$REPO"}
        - {name: RESTIC_PASSWORD, valueFrom: {secretKeyRef: {name: k8up-backup, key: RESTIC_PASSWORD}}}
        - {name: AWS_ACCESS_KEY_ID, valueFrom: {secretKeyRef: {name: k8up-backup, key: ACCESS_KEY_ID}}}
        - {name: AWS_SECRET_ACCESS_KEY, valueFrom: {secretKeyRef: {name: k8up-backup, key: SECRET_ACCESS_KEY}}}
      volumeMounts:
        - {name: config, mountPath: /config}
  volumes:
    - {name: config, persistentVolumeClaim: {claimName: audiobookshelf-config}}
EOF
```

Check the pod succeeded (`kubectl -n "$NS" get pod restore-absdb` shows
`Completed`; `kubectl -n "$NS" logs restore-absdb` shows no error). The swap
below also refuses to touch anything if the restored file is missing or empty.

Then swap the files. Move the old database **and its companion files** aside
together: audiobookshelf leaves SQLite in its default rollback-journal mode, and
an `absdatabase.sqlite-journal` left over from an interrupted write would be
rolled back into the restored database on first open, corrupting it. The
completed `restore-absdb` pod cannot be reused (pod specs are immutable), so
delete it and run the swap in a fresh pod:

```bash
kubectl -n "$NS" delete pod restore-absdb
kubectl -n "$NS" apply -f - <<'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: restore-absdb-swap
spec:
  restartPolicy: Never
  securityContext:
    runAsUser: 0
  containers:
    - name: swap
      image: alpine:3.22
      command: ["sh", "-euc"]
      args:
        - |
          cd /config
          # Refuse to touch anything unless the restored file is there and non-empty
          test -s absdatabase.sqlite.restored
          mkdir -p pre-restore
          for f in absdatabase.sqlite absdatabase.sqlite-journal absdatabase.sqlite-wal absdatabase.sqlite-shm; do
            if [ -e "$f" ]; then mv "$f" pre-restore/; fi
          done
          mv absdatabase.sqlite.restored absdatabase.sqlite
          ls -l absdatabase.sqlite pre-restore/
      volumeMounts:
        - {name: config, mountPath: /config}
  volumes:
    - {name: config, persistentVolumeClaim: {claimName: audiobookshelf-config}}
EOF
kubectl -n "$NS" wait --for=jsonpath='{.status.phase}'=Succeeded pod/restore-absdb-swap
kubectl -n "$NS" logs restore-absdb-swap
kubectl -n "$NS" delete pod restore-absdb-swap
```

Then `flux resume kustomization apps`, which brings audiobookshelf back to one
replica. The previous database stays in `pre-restore/` until you delete it.

### Clean up after a restore

```bash
kubectl -n "$NS" delete pod restore-verify --ignore-not-found
kubectl -n "$NS" delete restores.k8up.io "restore-$PVC" "restore-inplace-$PVC" --ignore-not-found
kubectl -n "$NS" delete pvc "restore-$PVC"
```

## Troubleshooting

### A PVC is not being backed up

- Check the annotation value is the string `"true"`, on the **PVC** (not the
  Deployment).
- Check the namespace has a `schedules.k8up.io/backup` and a `k8up-backup`
  Secret.
- Only `Bound` PVCs are considered. The operator log line for each PVC says why
  it was skipped.

### Backup job fails

```bash
kubectl -n "$NS" get jobs
kubectl -n "$NS" logs job/<backup-job-name>
```

- Credential or bucket errors: the R2 token must be scoped to
  `homelab-k8up-backups` with Object Read & Write.
- `permission denied` reading files: the job is not running as root — check
  `spec.podSecurityContext` on the Schedule.
- audiobookshelf: a failure in the backup command appears in the job log as the
  command's stderr (for example `SQLITE_CANTOPEN`). The dump runs in the
  audiobookshelf container itself; if that pod is not Running, no dump job is
  created and nothing fails (see "command-based SQLite dump" above).

### Repository locked

A job killed mid-run can leave a stale restic lock. Unlock it with restic
(using the environment from "Restore audiobookshelf's database") —
`restic unlock` removes only stale locks.

## Upgrades

Chart versions are pinned and updated by Renovate. The HelmRelease sets
`install.crds` and `upgrade.crds` to `CreateReplace`, because Helm installs
charts' `crds/` directory only on first install and never upgrades it; without
that, a chart bump would run a new operator against old CRDs. K8up's
[release notes](https://github.com/k8up-io/k8up/releases) flag breaking changes.

## References

- [K8up documentation](https://docs.k8up.io/)
- [K8up annotations reference](https://docs.k8up.io/k8up/references/annotations.html)
- [K8up application-aware backups](https://docs.k8up.io/k8up/how-tos/application-aware-backups.html)
- [K8up restore how-to](https://docs.k8up.io/k8up/how-tos/restore.html)
- [SQLite `VACUUM INTO`](https://www.sqlite.org/lang_vacuum.html#vacuuminto)
- [storage-migration-decision.md](storage-migration-decision.md) — why K8up, and why opt-in
