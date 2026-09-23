# Backup-failure alerting (Longhorn, CloudNativePG & K8up)

Alerting that fires when a backup **fails** or becomes **overdue**, for
Longhorn volume backups, CloudNativePG database backups and K8up volume
backups, with notifications delivered to Slack. Implements
[#234](https://github.com/aoshimash/homelab-k8s/issues/234); K8up coverage
added by [#340](https://github.com/aoshimash/homelab-k8s/issues/340) (see
[K8up](#k8up)).

## Why

Data integrity is the one guarantee this cluster does not compromise on. Backups
(Longhorn → R2, CloudNativePG → R2, K8up → R2) uphold it, but an unmonitored
backup is not a backup: a silent failure (bad credentials, R2 unreachable, a
broken schedule) would only surface at restore time — exactly when it is too
late.

## Architecture

Metrics already flow to Grafana Cloud (Alloy `remote_write`). There is no
in-cluster Prometheus/Alertmanager. So alert **rules live in Git as code** but are
**evaluated in Grafana Cloud**:

```
PrometheusRule (k8s/infrastructure/grafana-alloy/, Flux-managed)
   │  Alloy mimir.rules.kubernetes syncs them up
   ▼
Grafana Cloud Mimir ruler  ──evaluates against remote-written metrics──▶ alert fires
   │
   ▼
Grafana Cloud Mimir Alertmanager  ──routes──▶ Slack
   (config: grafana-cloud/alertmanager.yaml, applied via mimirtool)
```

Key pieces added:

| Piece | Location | Reconciled by |
|-------|----------|---------------|
| PrometheusRule CRD | `k8s/infrastructure/prometheus-operator-crds/` (Helm) | Flux — separate `infra-crds` Kustomization (`wait: true`) that `infrastructure` depends on, so the CRD exists before any PrometheusRule is applied |
| Backup metric scraping | `k8s/infrastructure/grafana-alloy/helmrelease.yaml` | Flux |
| K8up snapshot freshness metric | `k8s/infrastructure/kube-state-metrics/helmrelease.yaml` (`customResourceState`) | Flux |
| Ruler sync + RBAC | `helmrelease.yaml` (`mimir.rules.kubernetes`) + `rbac-prometheusrules.yaml` | Flux |
| Alert rules | `prometheusrule-backup-{longhorn,cnpg,k8up}.yaml` | Flux → Alloy → Grafana Cloud ruler |
| Slack routing | `grafana-cloud/alertmanager.yaml` | mimirtool (manual, **not** Flux) |

## Metrics used

Alloy scrapes these (kept to a tight allow-list to limit Grafana Cloud ingestion):

- **Longhorn** (`longhorn-manager` pods, `:9500`): `longhorn_backup_state`
  (`4` = Error), `longhorn_volume_last_backup_at` (unix ts of last successful
  backup, `0` if none), `longhorn_volume_robustness`.
- **CloudNativePG** (instance pods, `:9187`):
  `cnpg_collector_last_available_backup_timestamp`,
  `cnpg_collector_last_failed_backup_timestamp`,
  `cnpg_collector_first_recoverability_point`. *(Deprecated since CNPG 1.26 but
  functional with the in-core Barman Cloud R2 backups this cluster uses.)*
- **K8up operator** (Service `k8up-metrics` in `k8up`, `:8080`):
  `k8up_schedule_last_job_succeeded{namespace, schedule, jobType}` — `1` if
  the last job of that type a Schedule started succeeded, `0` if it failed.
- **K8up snapshots** (kube-state-metrics, from `snapshots.k8up.io`):
  `kube_customresource_k8up_snapshot_timestamp_seconds{namespace, path, snapshot}`
  — the time of each restic snapshot. It is forwarded by the existing
  kube-state-metrics scrape, which has no allow-list.

## Alert rules

| Alert | Condition | Notes |
|-------|-----------|-------|
| `LonghornBackupFailed` | `longhorn_backup_state == 4` for 5m | A backup is in Error state |
| `LonghornBackupOverdue` | no successful backup in >26h (`!= 0`) | Daily schedule 18:30 UTC + buffer; never-backed-up volumes excluded |
| `CNPGBackupFailed` | `last_failed > last_available` for 5m | A failure newer than the last good backup |
| `CNPGBackupOverdue` | no successful backup in >26h (`> 0`) | Daily schedule 18:00 UTC + buffer |
| `K8upJobFailed` | `k8up_schedule_last_job_succeeded == 0` for 5m | The last scheduled backup, check or prune job failed; clears when a later run of that type succeeds |
| `K8upBackupStale` | newest snapshot of a `(namespace, path)` older than 26h, for 15m | Daily schedule 19:00 UTC + buffer; every path, including backup-command dumps |
| `K8upSnapshotMetricsAbsent` | no snapshot series at all, for 1h | Guards `K8upBackupStale`, which cannot fire over a missing metric |

The 26h window (`93600s`) assumes the current daily schedules. Adjust the `expr`
thresholds if the backup cadence changes.

## K8up

K8up (restic) backs up opted-in PVCs daily at 19:00 UTC, one Schedule named
`backup` and one restic repository per namespace (see [k8up.md](k8up.md)). Its
rules are in `prometheusrule-backup-k8up.yaml`. They use two signals, because
neither is enough alone.

**Failure: `K8upJobFailed`.** The operator sets
`k8up_schedule_last_job_succeeded` when a job it started from a Schedule
finishes: `1` on success, `0` on failure, per `(namespace, schedule, jobType)`.
The value stays `0` until the next run of that type succeeds, like
`CNPGBackupFailed`. It covers the daily backup and the weekly prune and
`restic check`. Limits:

- Only Backup, Check and Prune objects labelled `k8up.io/schedule-name` are
  recorded. The operator adds that label to the ones a Schedule creates; an
  on-demand Backup without it does not move the metric.
- The value lives in operator memory. An operator restart drops the series
  until the next job of that type finishes, which resolves a firing alert
  without anything being fixed. For backups the freshness alert still catches
  it within a day; a failed weekly prune or check stays hidden until the next
  Sunday run.

**Freshness: `K8upBackupStale`.** Failure alerting cannot see a backup that
produces nothing and still succeeds. audiobookshelf's database is saved by a
`k8up.io/backupcommand` dump that runs in the audiobookshelf pod; if that pod is
not Running at 19:00 UTC, K8up creates no dump job and the Backup reports
success (see [k8up.md](k8up.md), "audiobookshelf: command-based SQLite dump").
Only the age of that path's newest snapshot shows the gap.

K8up has no last-success timestamp metric, so the age comes from its `Snapshot`
resources. After every successful backup or prune, the job mirrors the
repository's snapshot list into `snapshots.k8up.io` in the backed-up
namespace, one resource per restic snapshot with its `spec.date` and
`spec.paths`. K8up takes one snapshot per path (`/data/<pvc>` for a volume,
`/<namespace>-<container><ext>` for a dump). kube-state-metrics' CustomResourceState turns each into
`kube_customresource_k8up_snapshot_timestamp_seconds{namespace, path, snapshot}`,
and the rule takes `max by (namespace, path)`. Every backed-up path is covered
with no rule change when a volume or dump is added.

**Blind spot: `K8upSnapshotMetricsAbsent`.** `K8upBackupStale` over a missing
metric returns nothing and never fires. The absence alert fires when no
snapshot series has reached Grafana Cloud for 1h: kube-state-metrics down, its
CustomResourceState config broken, or no Snapshot resources left.

**Why the chart's rules are not used.** The K8up chart can render its own
`PrometheusRule` (`metrics.prometheusRule`), set explicitly to `enabled: false`
in `k8s/infrastructure/k8up/helmrelease.yaml`. Its rules do not fit this
cluster: `K8upResticErrors` needs `k8up_backup_restic_last_errors`, which K8up
only pushes to a Prometheus Pushgateway (none here); `K8upBackupNotRunning` is
`sum(rate(k8up_jobs_total[25h])) == 0 and on(namespace) k8up_schedules_gauge > 0`,
whose `sum()` drops `namespace`, so as written it can never match; the
`K8up<Type>Failed` rules need kube-state-metrics' `jobs` collector and a label
allow-list exposing `k8up.io/type` on `kube_job_labels`, neither enabled here. None checks freshness.

### Retired paths

A path that stops being backed up (a PVC renamed or removed from backups, a
dump removed) keeps alerting as `K8upBackupStale`. K8up's prune runs
`restic forget` with `keepDaily: 30` and no `--group-by`, so restic groups
snapshots by host and paths and keeps the last 30 days *that have snapshots*
for each group. A retired path's last 30 snapshots, and their Snapshot
resources, are therefore kept forever.

That is intended: the alert is the prompt to decide what to do with the dead
backup data. Once it is no longer needed:

1. Find the path's snapshots and forget them, with restic set up as in
   [k8up.md](k8up.md), "Restore audiobookshelf's database" (same image,
   repository and Secret keys):
   ```bash
   restic snapshots --path /data/<old-pvc>
   restic forget <snapshot-id> [<snapshot-id> ...]
   ```
   The data is freed by the next weekly prune.
2. The Snapshot resources are re-synced by the next successful backup or
   prune, which clears the alert. To clear it at once, delete them:
   `kubectl -n <namespace> delete snapshots.k8up.io <name> ...`
   (the resource name is the first 8 characters of the snapshot ID).

## Access policies & credentials (Grafana Cloud)

Two Grafana Cloud access-policy tokens are used, split by role so the long-lived
in-cluster token stays least-privileged. **Alloy never touches the Alertmanager
config**, so the ability to rewrite alert routing is kept out of the cluster.

| Token (policy) | Used by | Scopes | Stored where |
|----------------|---------|--------|--------------|
| `homelab-alloy-token` (`homelab-alloy`) | Alloy, in-cluster | `metrics:write`, `logs:write`, `rules:read`, `rules:write` | `GRAFANA_CLOUD_API_KEY` in `secret-grafana-cloud.sops.yaml` |
| `homelab-mimir-ops` token | operator, from a workstation | `alerts:write`, `alerts:read`, `rules:read` | password manager — **never** in the repo or cluster |

Why split: the Alloy token is decrypted into a DaemonSet pod env on every node;
granting it `alerts:write` would let any node rewrite/silence alert routing — the
opposite of what a backup-alerting feature should permit. `rules:read` is needed
alongside `rules:write` because `mimir.rules.kubernetes` reads the current ruler
state before applying diffs.

Secret-key ↔ Grafana Cloud value mapping (`secret-grafana-cloud.sops.yaml`):

| Secret key | Value | Where to find it |
|------------|-------|------------------|
| `GRAFANA_CLOUD_USER` | numeric metrics instance ID | Cloud Portal → Prometheus → Details |
| `GRAFANA_CLOUD_API_KEY` | `homelab-alloy-token` value | the token itself |
| `GRAFANA_CLOUD_PROMETHEUS_URL` | `…/api/prom/push` URL | Cloud Portal → Prometheus → Details |
| `GRAFANA_CLOUD_RULER_URL` | same host **without** `/api/prom/push` | derived from the push URL |

To **rotate** the Alloy token: create a new token under the `homelab-alloy`
policy, update `GRAFANA_CLOUD_API_KEY` in the secret, commit, let Flux reconcile.
The ops token rotates independently in the password manager.

## Setup / rotation (Grafana Cloud)

This procedure is reusable — follow it on first rollout and whenever the cluster
is rebuilt from scratch.

> [!IMPORTANT]
> On first rollout, do steps 1–3 **before merging** the PR. Until the ruler URL
> and `rules:*` scopes are in place, the ruler-sync component stays inactive (it
> is wired `optional: true`, so Alloy keeps collecting metrics/logs regardless)
> and no alerts are delivered.

### 1. Access policies & token scopes

- **`homelab-alloy` policy** (existing) → add `rules:read` and `rules:write` to
  its scopes, keeping `metrics:write` and `logs:write`. Its token
  (`homelab-alloy-token` = `GRAFANA_CLOUD_API_KEY`) inherits the new scopes, so
  **no secret change is needed**. (Access-policy tokens are prefixed `glc_`; if
  yours is a legacy `eyJ…` API key, instead create an access-policy token with
  these scopes and update the secret.)
- **`homelab-mimir-ops` policy** (new) → create it with `alerts:write`,
  `alerts:read`, `rules:read`, and a token used only from your workstation for
  the `mimirtool` steps below. Do not store this token in the repo or cluster.

### 2. Add the ruler URL to the SOPS secret

The ruler base URL is the Grafana Cloud Prometheus host **without** the
`/api/prom/push` suffix (e.g. `https://prometheus-prod-XX.grafana.net`). Find it
inside the existing credentials, then add a `GRAFANA_CLOUD_RULER_URL` key:

```bash
export SOPS_AGE_KEY_FILE=age.agekey

# Look up the existing remote_write URL to derive the ruler base URL:
sops -d k8s/infrastructure/grafana-alloy/secret-grafana-cloud.sops.yaml \
  | grep GRAFANA_CLOUD_PROMETHEUS_URL

# Edit the secret and add (strip the trailing /api/prom/push):
#   GRAFANA_CLOUD_RULER_URL: https://prometheus-prod-XX.grafana.net
sops k8s/infrastructure/grafana-alloy/secret-grafana-cloud.sops.yaml
```

### 3. Load the Slack Alertmanager config

The Mimir **Alertmanager is a separate endpoint** from the metrics/ruler host —
find its URL and instance ID in Cloud Portal → **Alertmanager → Details** (URL
like `https://alertmanager-prod-XX.grafana.net`; the instance ID **may differ**
from the metrics one). Using the `prometheus-prod-XX` host here returns a `404`.
See [`grafana-cloud/README.md`](../grafana-cloud/README.md). In short:

```bash
export SLACK_WEBHOOK_URL="https://hooks.slack.com/services/…"
envsubst < grafana-cloud/alertmanager.yaml > /tmp/am.yaml
mimirtool alertmanager load /tmp/am.yaml \
  --address="https://alertmanager-prod-XX.grafana.net" \
  --id="<alertmanager-instance-id>" --key="<homelab-mimir-ops token>"
rm -f /tmp/am.yaml
```

### 4. Merge & reconcile

Merge the PR. Flux installs the CRD, reconfigures Alloy, and applies the
PrometheusRule resources. Confirm:

```bash
# CRD present
kubectl get crd prometheusrules.monitoring.coreos.com

# Rules applied in-cluster
kubectl -n monitoring get prometheusrules

# Alloy healthy and the ruler-sync component up (no errors)
kubectl -n monitoring logs ds/grafana-alloy | grep -i "mimir.rules" 

# Rules visible in Grafana Cloud
mimirtool rules list --address="$MIMIR_ADDRESS" --id="$MIMIR_TENANT_ID" --key="$MIMIR_API_KEY"
```

In Grafana Cloud, the metrics should be queryable (e.g.
`longhorn_volume_last_backup_at`, `cnpg_collector_last_available_backup_timestamp`)
and the alert rules should appear under Alerting → Alert rules (data-source-managed).

## Verifying alert delivery (Acceptance Criterion #3)

> *"The alert reaches a notification destination, verified by deliberately
> inducing a backup failure (e.g. invalid R2 credentials)."*

This is a post-merge, live-cluster step — it cannot be done from the PR alone.

### CloudNativePG (recommended — easiest to induce and revert)

> [!WARNING]
> The CNPG instance **caches the R2 credential**. Restoring the secret alone does
> NOT recover backups — the running instance keeps using the stale value and WAL
> archiving stays broken (`SignatureDoesNotMatch`). You MUST restart the instance
> (step 4) to reload it. Skipping the restart leaves the cluster's backups broken
> after the test.

1. **Stop Flux from reverting the secret mid-test:**
   ```bash
   flux suspend kustomization configs
   ```
2. **Corrupt the R2 secret** so the next backup fails:
   ```bash
   kubectl -n postgres patch secret postgres-r2-credentials --type=json \
     -p="[{\"op\":\"replace\",\"path\":\"/data/ACCESS_SECRET_KEY\",\"value\":\"$(printf INVALID | base64)\"}]"
   ```
3. **Trigger an on-demand backup** (it will fail):
   ```bash
   kubectl -n postgres create -f - <<'EOF'
   apiVersion: postgresql.cnpg.io/v1
   kind: Backup
   metadata:
     generateName: verify-alert-
     namespace: postgres
   spec:
     cluster:
       name: postgres-cluster
   EOF
   ```
   This advances `cnpg_collector_last_failed_backup_timestamp` past
   `cnpg_collector_last_available_backup_timestamp`. After the rule's `for: 5m`,
   `CNPGBackupFailed` fires → Slack. **Confirm the firing (🔴) message.**
4. **Restore the credential AND restart the instance** (the restart is mandatory):
   ```bash
   flux resume kustomization configs
   flux reconcile kustomization configs --with-source   # re-applies the real secret
   kubectl -n postgres delete pod postgres-cluster-1     # reload credential (brief restart)
   # wait for recovery — need Ready=True and ContinuousArchiving=True:
   kubectl -n postgres get cluster postgres-cluster \
     -o jsonpath='{range .status.conditions[*]}{.type}={.status}{"\n"}{end}'
   ```
5. **Trigger a fresh, successful backup** to resolve the alert (delete any stuck
   `verify-*` backups first):
   ```bash
   kubectl -n postgres create -f - <<'EOF'
   apiVersion: postgresql.cnpg.io/v1
   kind: Backup
   metadata:
     generateName: verify-resolve-
     namespace: postgres
   spec:
     cluster:
       name: postgres-cluster
   EOF
   ```
   Once it completes, `last_available` exceeds `last_failed`, `CNPGBackupFailed`
   clears, and a resolved (🟢) notification is sent. **Confirm the resolved
   message and `LastBackupSucceeded=True`.**

#### Observing without the `metrics:read` scope

Querying Grafana Cloud metrics needs `metrics:read`, which the in-cluster
`homelab-alloy` token does not have. To verify from the operator side anyway:

- **Raw backup timestamps** — read them straight from the instance:
  ```bash
  kubectl -n postgres port-forward postgres-cluster-1 9187:9187 &
  curl -s localhost:9187/metrics | grep '^cnpg_collector_last_'
  ```
- **Alert state in the ruler** — `rules:read` (which `homelab-alloy` has) is enough:
  ```bash
  curl -s -u "<instance-id>:<token>" \
    "https://prometheus-prod-XX.grafana.net/api/prom/api/v1/rules?type=alert"
  # look for CNPGBackupFailed .state: inactive → pending → firing → inactive
  ```

### Longhorn

Induce a failure by temporarily setting an invalid backup target secret
(`longhorn-r2-credentials`) or an unreachable `backupTarget`, then trigger the
`backup-daily` recurring job (or create a manual backup) from the Longhorn UI.
`longhorn_backup_state` goes to `4` (Error) → `LonghornBackupFailed` fires →
Slack. Revert afterwards.

> [!CAUTION]
> Always restore the real credentials and confirm a subsequent backup succeeds.
> Leaving broken credentials in place defeats the purpose of the backups.

### K8up

**`K8upJobFailed`.** Induce a failure with an on-demand Backup that uses a
wrong repository password. This changes no credentials, Secret or Schedule, so
Flux does not need suspending and nothing has to be restored afterwards:
`restic init` refuses a repository that already exists, and a wrong password
cannot open it, so nothing is written to it. The Backup carries the
`k8up.io/schedule-name: backup` label, so the operator records its outcome in
`k8up_schedule_last_job_succeeded` like a scheduled run.

1. **Start the failing backup** (vikunja has no backup command, so only the
   volume backup runs):
   ```bash
   NS=vikunja
   kubectl -n "$NS" get schedules.k8up.io backup -o json \
     | jq '{apiVersion: "k8up.io/v1", kind: "Backup",
            metadata: {generateName: "verify-alert-",
                       labels: {"k8up.io/schedule-name": "backup"}},
            spec: {backend: (.spec.backend | .repoPasswordSecretRef.key = "ACCESS_KEY_ID"),
                   podSecurityContext: .spec.podSecurityContext}}' \
     | kubectl -n "$NS" create -f -
   ```
   The job's pods fail because restic cannot open the repository
   (`wrong password or no key found`). The Job is
   retried up to the operator's backoff limit (6 by default), so it takes about
   10 minutes to be marked failed. The metric then drops to `0`, and after the
   rule's `for: 5m`, `K8upJobFailed` fires → Slack. **Confirm the firing (🔴)
   message.**
2. **Resolve it** with the same Backup and the real password:
   ```bash
   kubectl -n "$NS" get schedules.k8up.io backup -o json \
     | jq '{apiVersion: "k8up.io/v1", kind: "Backup",
            metadata: {generateName: "verify-resolve-",
                       labels: {"k8up.io/schedule-name": "backup"}},
            spec: {backend: .spec.backend,
                   podSecurityContext: .spec.podSecurityContext}}' \
     | kubectl -n "$NS" create -f -
   ```
   When it completes the metric returns to `1`, `K8upJobFailed` clears, and a
   resolved (🟢) notification is sent. **Confirm the resolved message.**
3. **Clean up** the two Backups (their Jobs and pods go with them):
   ```bash
   kubectl -n "$NS" get backups.k8up.io -o name | grep '/verify-' \
     | xargs kubectl -n "$NS" delete
   ```

**`K8upBackupStale`.** Waiting 26h for a real gap is not practical, so check
that the expression evaluates against real data instead. In Grafana Cloud
Explore:

```promql
time() - max by (namespace, path) (kube_customresource_k8up_snapshot_timestamp_seconds)
```

It should return one series per backed-up path, including
`/audiobookshelf-audiobookshelf.sqlite`, each with the age in seconds of the
newest snapshot (under `86400` plus the backup's run time, right after the
daily backup). Compare with
`kubectl get snapshots.k8up.io -A -o custom-columns='NS:.metadata.namespace,DATE:.spec.date,PATHS:.spec.paths'`.

## Troubleshooting

- **No rules in Grafana Cloud** → check Alloy logs for `mimir.rules.kubernetes`
  errors (usually a missing `GRAFANA_CLOUD_RULER_URL`, or the `homelab-alloy`
  policy lacking `rules:read`/`rules:write`). The component is wired
  `optional: true`, so a missing URL silently disables sync without breaking Alloy.
- **`K8upSnapshotMetricsAbsent` firing** → check
  `kubectl get snapshots.k8up.io -A` lists snapshots, then the
  kube-state-metrics pod log for CustomResourceState errors (a denied
  `list snapshots.k8up.io` means the `rbac.extraRules` entry is missing).
- **Rules present but no Slack** → the Alertmanager config was not loaded; re-run
  step 3. Verify with `mimirtool alertmanager get`.
- **Alert never fires** → confirm the underlying metric exists in Grafana Cloud
  (Explore). If absent, the scrape is not reaching the pod — check the pod
  labels/ports against the `discovery.kubernetes` selectors in the Alloy config.
- **`mimirtool alertmanager load` returns `404 requested resource not found`** →
  you used the metrics/ruler host. The Alertmanager is a separate endpoint
  (`alertmanager-prod-XX`, see step 3) with its own instance ID.
- **`mimirtool` returns `401 invalid authentication credentials`** → check the
  `--id`/`--key`. A common cause when deriving `--id` from `sops -d` output is
  that YAML serialization wraps numeric instance IDs in quotes — strip them
  (`tr -d '[:space:]"'`) or the basic-auth username becomes `"123"` not `123`.
  Otherwise verify the access policy's realm includes this stack and the scopes
  (`rules:*` for the ruler, `alerts:*` for the Alertmanager).
