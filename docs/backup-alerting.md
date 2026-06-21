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
| PrometheusRule CRD | `k8s/infrastructure/prometheus-operator-crds/` (Helm) | Flux |
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

## Rollout (one-time)

> [!IMPORTANT]
> Do these **before merging** the PR. Until the secret key and token scopes are
> in place, the ruler-sync component stays inactive (it is wired with
> `optional: true`, so Alloy keeps collecting metrics/logs regardless), and no
> alerts are delivered.

### 1. Grafana Cloud token scopes

The existing `GRAFANA_CLOUD_API_KEY` (used for `remote_write`) is reused for
ruler sync. Ensure its access policy includes the **`rules:write`** scope (and
`alerts:write` for the Alertmanager step). If not, edit the access policy in
Grafana Cloud → Account → Access Policies, or create a new token and update the
secret.

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

See [`grafana-cloud/README.md`](../grafana-cloud/README.md). In short:

```bash
export SLACK_WEBHOOK_URL="https://hooks.slack.com/services/…"
envsubst < grafana-cloud/alertmanager.yaml > /tmp/am.yaml
mimirtool alertmanager load /tmp/am.yaml \
  --address="https://prometheus-prod-XX.grafana.net" \
  --id="<instance-id>" --key="<token-with-alerts:write>"
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

1. Temporarily break the R2 credentials so the next scheduled/triggered backup
   fails. Either trigger an on-demand backup against a cluster with bad creds, or
   point the credentials secret at an invalid key:
   ```bash
   # Trigger an on-demand backup to get a fresh attempt:
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
   To force a failure, first corrupt `ACCESS_SECRET_KEY` in the
   `postgres-r2-credentials` secret (note the original value to restore it).
2. The failed backup advances `cnpg_collector_last_failed_backup_timestamp` past
   `cnpg_collector_last_available_backup_timestamp`. Within the rule's `for: 5m`,
   `CNPGBackupFailed` fires and Grafana Cloud routes it to Slack.
3. **Confirm the Slack message arrives.**
4. **Revert** the credentials, re-trigger a backup, and confirm a successful
   backup clears the alert (a resolved notification is sent).

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
  errors (usually a missing `GRAFANA_CLOUD_RULER_URL` or a token without
  `rules:write`). The component is wired `optional: true`, so a missing URL
  silently disables sync without breaking Alloy.
- **Rules present but no Slack** → the Alertmanager config was not loaded; re-run
  step 3. Verify with `mimirtool alertmanager get`.
- **Alert never fires** → confirm the underlying metric exists in Grafana Cloud
  (Explore). If absent, the scrape is not reaching the pod — check the pod
  labels/ports against the `discovery.kubernetes` selectors in the Alloy config.
