# Renovate Operator

The [mogenius renovate-operator](https://github.com/mogenius/renovate-operator)
runs self-hosted Renovate on this cluster. It schedules Renovate as Kubernetes
Jobs, discovers repositories, accepts GitHub webhooks, and has a web UI.
It replaces the hosted Mend Renovate app for the owner's personal repositories
([#403](https://github.com/aoshimash/homelab-k8s/issues/403)).

This page covers the operator itself. How Renovate is configured for this
repository is in [renovate.md](renovate.md).

## Current Configuration

| Item | Value |
|------|-------|
| Chart | `renovate-operator` from `oci://ghcr.io/mogenius/helm-charts`, pinned in `k8s/infrastructure/renovate-operator/helmrelease.yaml` |
| Image | `ghcr.io/mogenius/renovate-operator`, the chart's `appVersion` (moves with the chart) |
| Namespace | `renovate-operator` |
| CRDs | Rendered by the chart (`crd.mode: template`), upgraded with each chart bump |
| UI | `https://renovate.<tailnet>.ts.net`, tailnet-only (`k8s/configs/renovate/ingress.yaml`) |
| UI login | OIDC through [tsidp](tsidp.md). Client ID, client secret and session key in `secret-oidc.sops.yaml` |
| Access control | Off (`authorization.enabled: false`): anyone who signs in through tsidp has full access |
| GitHub identity | A private GitHub App owned by the personal account. App ID and private key in `k8s/configs/renovate/secret-github-app.sops.yaml` |
| Jobs | `githubappinstallationsets.renovate-operator.mogenius.com` `personal` generates one `RenovateJob` per installation from the template `installation-defaults` |
| Renovate image | `ghcr.io/renovatebot/renovate`, `tag@sha256` in `k8s/configs/renovate/renovatejobtemplate.yaml` |
| Schedule | Hourly (`H * * * *`, UTC, a per-job minute), plus GitHub webhooks |
| Webhook | `https://renovate-webhook.<tailnet>.ts.net/webhook/v1/github`, the cluster's only internet-facing endpoint (Tailscale Funnel) |
| Finished Jobs | Kept for a day (`config.jobTTLSecondsAfterFinished`), then garbage-collected with their logs |
| Alert | `RenovateRunFailed` (`k8s/infrastructure/grafana-alloy/prometheusrule-renovate.yaml`) |
| Valkey / S3 | Not used. Logs of past runs are not stored; the Job pods' logs are the record |

Why it is wired this way:

- **The UI Ingress is not the chart's.** The chart's Ingress writes a `host`
  rule, and its redirect-URL detection assumes one. The Tailscale Ingress names
  its hostname in `tls.hosts` instead, and it uses the `ingress-proxies`
  ProxyGroup from the `configs` layer. So the chart's Ingress is off, the
  Ingress lives in `k8s/configs/renovate/`, and `auth.oidc.redirectUrl` is set
  explicitly.
- **A fixed session key.** Without `sessionSecretKey` the operator makes a new
  session key at every start, and every restart logs everyone out.
- **No per-user access rules.** Naming users in `authorization.defaults` would
  put login names into this public repository. Signing in through tsidp already
  requires tailnet membership.

## Custom Resources

The chart installs these CRDs. Refer to them by their full names, since the
short kind names are generic:

| Resource | Purpose |
|----------|---------|
| `renovatejobs.renovate-operator.mogenius.com` | One Renovate setup: schedule, credentials, discovery, parallelism |
| `renovatejobtemplates.renovate-operator.mogenius.com` | Settings that generated `RenovateJob`s inherit |
| `clusterrenovatejobtemplates.renovate-operator.mogenius.com` | The cluster-scoped form of the template |
| `githubappinstallationsets.renovate-operator.mogenius.com` | Generates one `RenovateJob` per installation of a GitHub App |
| `renovateprojects.renovate-operator.mogenius.com` | The operator's per-repository status records |

```bash
kubectl get crd | grep renovate-operator.mogenius.com
kubectl -n renovate-operator get githubappinstallationsets.renovate-operator.mogenius.com
kubectl -n renovate-operator get renovatejobs.renovate-operator.mogenius.com
```

## Which Repositories Renovate Runs On

Only the GitHub App's repository selection decides it. The App is installed on
the personal account with **Only select repositories**, and nothing in this
repository filters further.

- **Add a repository:** GitHub → Settings → Applications → the App →
  Configure → add the repository. GitHub sends an `installation_repositories`
  event to the operator's webhook, and the installation set starts discovery at
  once, so the repository appears in the UI within minutes. A repository with
  no Renovate config gets an onboarding PR when Renovate finds dependencies in
  it; with none, nothing happens (Renovate's default for discovered
  repositories).
- **Remove a repository:** deselect it the same way. It drops out at the next
  discovery.
- **Never select a repository in both this App and the hosted Mend app.** Two
  bots on one repository open duplicate PRs.

Only an installation set reacts to `installation_repositories` with an
immediate discovery; a hand-written `RenovateJob` would wait for its next
scheduled discovery. That is why the jobs are generated rather than written.

## GitHub App

A private App can only be installed on the account that owns it, which keeps
Renovate to the personal account.

| Setting | Value |
|---------|-------|
| Visibility | Private ("Only on this account") |
| Webhook URL | `https://renovate-webhook.<tailnet>.ts.net/webhook/v1/github` |
| Webhook secret | `secret` in `k8s/configs/renovate/secret-github-app-webhook.sops.yaml` |
| Events | Issues, Pull request (`installation` and `installation_repositories` arrive without a subscription) |
| Repository permissions | The table in Renovate's GitHub platform docs, "Running as a GitHub App": Administration read; Checks, Commit statuses, Contents, Issues, Merge queues, Pull requests, Workflows read and write; Dependabot alerts and Metadata read |

The private key and App ID live in `secret-github-app.sops.yaml` as `PEM` and
`APP_ID`. The operator mints short-lived installation tokens from them; no
long-lived token exists. To rotate the key, generate a new one in the App
settings, re-encrypt the Secret, merge, then delete the old key on GitHub.

The same webhook secret serves two checks: the installation set verifies
GitHub's `X-Hub-Signature-256` on installation events, and every generated job
verifies it on PR and issue events (`webhook.authentication` in the template).

## Webhook Exposure

The webhook Ingress (`k8s/configs/renovate/ingress-webhook.yaml`) is the only
path from the internet into the cluster. It is limited three ways:

1. **Only `/webhook/v1/github` is routed.** The Ingress has no default
   backend, so the public hostname serves nothing else. The operator's
   `/webhook/v1/schedule` API (which can start runs) and the UI are not
   reachable from the internet.
2. **Only the webhook listener is behind it.** It points at the Service's
   port 8082. The UI (8081) has its own tailnet-only Ingress, and
   `webhook.unifiedWebhookHost` stays `false` so the operator does not serve
   webhooks on the UI's host either.
3. **Requests are signed.** Webhook authentication is on for every job, so an
   unsigned or wrongly signed delivery is rejected.

Funnel needs a standalone proxy: the Tailscale operator supports Funnel only on
per-Ingress proxies, not on a ProxyGroup (as of v1.104.1). The proxy is tagged
`tag:k8s-funnel` so that only it may use Funnel. The tailnet policy (edited in
the Tailscale admin console, not in Git) needs:

```jsonc
{
  "tagOwners": {
    // The operator creates the proxy device with this tag.
    "tag:k8s-funnel": ["tag:k8s-operator"]
  },
  "nodeAttrs": [
    // Funnel for the webhook proxy only. autogroup:member does not cover
    // tagged devices, so this grant is required.
    { "target": ["tag:k8s-funnel"], "attr": ["funnel"] }
  ]
}
```

## Alerting

With the hosted Mend app, failed runs showed up in its portal. Here,
`RenovateRunFailed` fires when the operator's `renovate_operator_run_failed`
gauge is 1 for a project, i.e. its last run failed. Grafana Alloy scrapes the
operator pod's metrics port (8080) directly and forwards only that metric.

To see why a run failed: open the project in the UI, or
`kubectl -n renovate-operator get jobs` and read the job pod's log. Finished
Jobs are kept for a day.

## UI Login Through tsidp

### Registering the Client

The client is registered by hand in tsidp's admin UI
([tsidp.md](tsidp.md#registering-an-oidc-client)):

- Redirect URI: `https://renovate.<tailnet>.ts.net/auth/callback`. It must
  match `auth.oidc.redirectUrl` in the HelmRelease exactly.
- Store the client ID and secret as `client-id` and `client-secret` in
  `k8s/infrastructure/renovate-operator/secret-oidc.sops.yaml`, next to a random
  `session-secret` (`openssl rand -hex 32`).

The HelmRelease reads `client-id` through `valuesFrom`. The chart reads the
other two keys from the same Secret.

### Reaching tsidp from the Pod

The operator calls tsidp from its pod for discovery, the token exchange and
the signing keys. Pods are not on the tailnet and cannot resolve `*.ts.net`,
but the node routes the tailnet range. The HelmRelease therefore adds a
`hostAliases` entry that maps `idp.<tailnet>.ts.net` to tsidp's tailnet IP,
through a `postRenderers` patch, because the chart has no `hostAliases` value.
This is the same mechanism Outline uses, and it breaks for the same reasons
([outline.md](outline.md#reaching-tsidp-from-the-pod)). If tsidp re-registers
with a new IP, follow the recovery steps in
[tsidp.md](tsidp.md#backup-and-recovery), which list every place that holds it.

### Checking It

```bash
flux get hr -n renovate-operator renovate-operator
kubectl -n renovate-operator get deploy,svc,ingress
kubectl -n renovate-operator logs deploy/renovate-operator | grep -i oidc
```

Then, from a tailnet device:

- `https://renovate.<tailnet>.ts.net` redirects to tsidp, and the UI appears
  only after signing in.
- The same hostname does not resolve from outside the tailnet.

## Upgrading

Renovate proposes chart bumps and bumps of the Renovate image in the template
(the `kubernetes` manager reads its `image:` line). The operator image is the
chart's `appVersion`, so the chart PR is the image PR too. Before merging a major version, read the
upstream migration notes (`docs/migration/` in the operator repository): v5 to
v6 changed the policy engine and access control.
