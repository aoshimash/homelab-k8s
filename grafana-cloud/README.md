# Grafana Cloud configuration (managed as code, applied via mimirtool)

Grafana Alloy syncs in-cluster `PrometheusRule` resources to the Grafana Cloud
Mimir ruler automatically (via the `mimir.rules.kubernetes` component). However,
Alloy does **not** sync the Mimir Alertmanager configuration — the routing that
delivers fired alerts to Slack. That part lives here and is applied out-of-band
with [`mimirtool`](https://grafana.com/docs/mimir/latest/manage/tools/mimirtool/).

This directory is intentionally **outside `k8s/`** so Flux does not try to
reconcile it as a Kubernetes manifest.

## Files

- `alertmanager.yaml` — Mimir Alertmanager config: routes all alerts to a Slack
  receiver. The webhook URL is injected at apply time (never committed).

## Prerequisites

- `mimirtool` installed (`brew install mimirtool` or download from the
  [Grafana Mimir releases](https://github.com/grafana/mimir/releases)).
- The `homelab-mimir-ops` access-policy token (`alerts:write` + `alerts:read`) —
  the operator token, separate from the in-cluster Alloy token. See the access
  policy table in `docs/backup-alerting.md`.
- A Slack [incoming webhook](https://api.slack.com/messaging/webhooks) URL for
  the target channel.

## Apply

> [!IMPORTANT]
> The Mimir **Alertmanager is a separate endpoint** from the metrics/ruler one.
> Use the Alertmanager URL and instance ID from Cloud Portal → **Alertmanager →
> Details** — the URL looks like `https://alertmanager-prod-XX.grafana.net` and
> the instance ID **may differ** from the metrics instance ID. Using the
> `prometheus-prod-XX` host here returns `404 requested resource not found`.

```bash
# Alertmanager endpoint + its own instance ID (Cloud Portal → Alertmanager → Details).
export MIMIR_AM_ADDRESS="https://alertmanager-prod-XX.grafana.net"
export MIMIR_AM_ID="<alertmanager-instance-id>"
export MIMIR_API_KEY="<homelab-mimir-ops token: alerts:write + alerts:read>"
export SLACK_WEBHOOK_URL="https://hooks.slack.com/services/XXXX/YYYY/ZZZZ"

# Render the webhook into the config and load it.
envsubst < alertmanager.yaml > /tmp/alertmanager.rendered.yaml
mimirtool alertmanager load /tmp/alertmanager.rendered.yaml \
  --address="${MIMIR_AM_ADDRESS}" --id="${MIMIR_AM_ID}" --key="${MIMIR_API_KEY}"
rm -f /tmp/alertmanager.rendered.yaml

# Verify what is currently loaded:
mimirtool alertmanager get \
  --address="${MIMIR_AM_ADDRESS}" --id="${MIMIR_AM_ID}" --key="${MIMIR_API_KEY}"
```

The alert **rules** themselves are NOT pushed from here — Alloy syncs them from
the `PrometheusRule` resources in `k8s/infrastructure/grafana-alloy/`. See
`docs/backup-alerting.md` for the full architecture and verification runbook.
