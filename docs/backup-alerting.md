# Backup-failure alerting (Longhorn & CloudNativePG)

Alerting that fires when a backup **fails** or becomes **overdue**, for both
Longhorn volume backups and CloudNativePG database backups, with notifications
delivered to Slack. Implements [#234](https://github.com/aoshimash/homelab-k8s/issues/234).

## Why

Data integrity is the one guarantee this cluster does not compromise on. Backups
(Longhorn → R2, CloudNativePG → R2) uphold it, but an unmonitored backup is not a
backup: a silent failure (bad credentials, R2 unreachable, a broken schedule)
would only surface at restore time — exactly when it is too late.

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
| Ruler sync + RBAC | `helmrelease.yaml` (`mimir.rules.kubernetes`) + `rbac-prometheusrules.yaml` | Flux |
| Alert rules | `prometheusrule-backup-{longhorn,cnpg}.yaml` | Flux → Alloy → Grafana Cloud ruler |
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

## Alert rules

| Alert | Condition | Notes |
|-------|-----------|-------|
| `LonghornBackupFailed` | `longhorn_backup_state == 4` for 5m | A backup is in Error state |
| `LonghornBackupOverdue` | no successful backup in >26h (`!= 0`) | Daily schedule 02:00 + buffer; never-backed-up volumes excluded |
| `CNPGBackupFailed` | `last_failed > last_available` for 5m | A failure newer than the last good backup |
| `CNPGBackupOverdue` | no successful backup in >26h (`> 0`) | Daily schedule 21:00 UTC + buffer |

The 26h window (`93600s`) assumes the current daily schedules. Adjust the `expr`
thresholds if the backup cadence changes.

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

## Troubleshooting

- **No rules in Grafana Cloud** → check Alloy logs for `mimir.rules.kubernetes`
  errors (usually a missing `GRAFANA_CLOUD_RULER_URL`, or the `homelab-alloy`
  policy lacking `rules:read`/`rules:write`). The component is wired
  `optional: true`, so a missing URL silently disables sync without breaking Alloy.
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
