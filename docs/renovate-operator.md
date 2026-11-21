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

None of them is instantiated yet. The GitHub App, the jobs and the webhook come
with [#405](https://github.com/aoshimash/homelab-k8s/issues/405).

```bash
kubectl get crd | grep renovate-operator.mogenius.com
```

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
with a new IP, update both places.

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

Renovate proposes chart bumps. The operator image is the chart's `appVersion`,
so the chart PR is the image PR too. Before merging a major version, read the
upstream migration notes (`docs/migration/` in the operator repository): v5 to
v6 changed the policy engine and access control.
