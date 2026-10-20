# Outline - Knowledge Base

[Outline](https://github.com/outline/outline) is the knowledge hub for notes
and documents, introduced by
[#378](https://github.com/aoshimash/homelab-k8s/issues/378) as part of
[#376](https://github.com/aoshimash/homelab-k8s/issues/376). People use it in
the browser; Claude Code on a tailnet device reads and writes it through
Outline's built-in MCP endpoint. It is reachable only on the tailnet, and login
is through [tsidp](tsidp.md).

## Current Configuration

| Item | Value |
|------|-------|
| Manifests | `k8s/apps/outline/` |
| Image | `outlinewiki/outline`, pinned `tag@sha256` in `deployment.yaml` |
| URL | `https://outline.tail19032f.ts.net` (Tailscale ProxyGroup Ingress) |
| MCP endpoint | `https://outline.tail19032f.ts.net/mcp` |
| Login | OIDC via tsidp (`https://idp.tail19032f.ts.net`) |
| Database | `outline` on the shared CloudNativePG cluster, role `outline` |
| Attachments | `FILE_STORAGE=local`, `/var/lib/outline/data` on PVC `outline-data` (local-path), K8up-backed |
| Redis | Sidecar in the same pod, no persistence (queues and caches only) |
| Pod user | 1001 (the image's `nodejs` user) |

Non-secret settings are in `configmap.yaml`. The Secret `outline`
(`secret-outline.sops.yaml`) holds `SECRET_KEY`, `UTILS_SECRET` and the tsidp
client (`OIDC_CLIENT_ID`, `OIDC_CLIENT_SECRET`). The Secret `outline-db`
(`secret-db.sops.yaml`) holds the database connection. Outline reads them as
the separate `DATABASE_HOST`/`PORT`/`NAME`/`USER`/`PASSWORD` variables, so no
URL with an embedded password is stored.

**Attachments are on a PVC, not R2.** The parent issue chose local storage
(#376, Design Decisions) because Outline's file-storage documentation listed
Cloudflare R2 as not compatible with its S3 mode. Outline (v1.11.0 included)
has `AWS_S3_UPLOAD_METHOD=put` for providers such as R2 that lack presigned
POST (`.env.sample`), so moving attachments to R2 would now be a new decision
rather than a known incompatibility.

### Access flow

```
Browser / Claude Code (tailnet device)
  → ingress-proxies (ProxyGroup, TLS) → Service outline:3000 → Pod
```

TLS ends at the ProxyGroup, which forwards plain HTTP, so `FORCE_HTTPS` is
`false`. Leaving it at Outline's default `true` would also redirect the
kubelet's HTTP probes.

### Reaching tsidp from the pod

Outline's server talks to tsidp itself, not only through the browser:

- at startup it fetches `https://idp.tail19032f.ts.net/.well-known/openid-configuration`
  (`OIDC_ISSUER_URL`) and mounts the login routes only if that succeeds;
- at each login it calls the token endpoint and the userinfo endpoint.

It does not fetch the JWKS. As of v1.11.0 it decodes the ID token without
verifying it and takes the user from `/userinfo` (`plugins/oidc/server/auth/oidcRouter.ts`).

Pods are not on the tailnet and cannot resolve `*.ts.net`, but **the node is**:
Talos runs `tailscaled` from the `siderolabs/tailscale` extension, and it routes
the tailnet range. Pod traffic to a tailnet IP leaves through the node and
reaches tsidp. So only the name has to be resolved, and the Deployment does
that with `hostAliases`:

```yaml
hostAliases:
  - ip: 100.89.177.82          # tsidp's tailnet IP
    hostnames:
      - idp.tail19032f.ts.net
```

TLS still verifies against tsidp's own certificate for
`idp.tail19032f.ts.net`, so nothing is weakened. Checked on 2026-10-20 from a
throwaway pod: `curl --resolve idp.tail19032f.ts.net:443:100.89.177.82
https://idp.tail19032f.ts.net/.well-known/openid-configuration` returned 200.

What this depends on, and what breaks it:

| Dependency | If it changes |
|------------|---------------|
| tsidp's tailnet IP (`100.89.177.82`) | It changes only if tsidp's state is lost and the node re-registers ([tsidp.md](tsidp.md#backup-and-recovery)). Update `hostAliases` in `deployment.yaml` to the new IP (`tailscale ip -4 idp`). |
| The node's `tailscaled` | If it is down, pods cannot reach any tailnet address. Outline keeps serving logged-in users but new logins fail, and a restarting pod fails its OIDC discovery. |
| Tailnet policy | The back-channel requests arrive at tsidp from the node, whose tailnet identity is `tag:homelab`. Today the allow-all ACL permits it. If that rule is removed, add a grant from `tag:homelab` to `tag:tsidp` on `tcp:443` (see [tsidp.md](tsidp.md#tailnet-policy)). |

tsidp authenticates the token request by the client secret, not by the calling
node. Its "tagged nodes not supported" checks apply to the user who signed in
through the browser, not to the caller of `/token` or `/userinfo`
(`server/token.go` and `server/userinfo.go`, as of v0.0.17), so a call from a
tagged node is fine.

**Alternatives not taken.** A Tailscale operator egress Service
(`tailscale.com/tailnet-fqdn`) gives a proxy with the `tag:k8s` identity, but
the pod would still need the FQDN to resolve to it. The operator's `DNSConfig`
nameserver handles that only after a `ts.net` stub zone is added to CoreDNS,
and CoreDNS's ConfigMap is a Talos-managed manifest (its field manager is
`talos`). The proxy's and the nameserver's ClusterIPs are assigned when the
operator creates their Services, so `hostAliases` cannot pin them either.
That is two new moving parts plus a CoreDNS change, for a path the node
already provides. Revisit if a second node joins without the Tailscale
extension ([#307](https://github.com/aoshimash/homelab-k8s/issues/307)) or if
the egress source must be distinguishable from the node.

## First Deployment

### 1. Register the tsidp client (before merging)

In the tsidp admin UI (`https://idp.tail19032f.ts.net/`, see
[tsidp.md](tsidp.md#registering-an-oidc-client)), create a client:

| Field | Value |
|-------|-------|
| Name | `outline` |
| Redirect URI | `https://outline.tail19032f.ts.net/auth/oidc.callback` |

### 2. Put the client into the Secret

The PR that introduced Outline (#378) committed placeholder values for
`OIDC_CLIENT_ID` and `OIDC_CLIENT_SECRET`. Set the real ones from the
repository root, without the values reaching the shell history or the process
list. The same commands replace the client later, e.g. after a tsidp
recovery:

```bash
export SOPS_AGE_KEY_FILE=age.agekey
F=k8s/apps/outline/app/secret-outline.sops.yaml
read -rs 'ID?client ID: ' && echo
printf '"%s"' "$ID" | sops set --value-stdin "$F" '["stringData"]["OIDC_CLIENT_ID"]'
read -rs 'SECRET?client secret: ' && echo
printf '"%s"' "$SECRET" | sops set --value-stdin "$F" '["stringData"]["OIDC_CLIENT_SECRET"]'
unset ID SECRET
```

(`read -rs 'VAR?prompt'` is zsh; in bash use `read -rsp 'prompt' VAR`.) Commit
and push the file. The placeholders do not stop Outline from starting, but
every login fails until they are replaced.

### 3. Merge and check

After Flux reconciles `configs` (database and role) and `apps`:

```bash
kubectl -n outline get pods            # 2/2 Running
kubectl -n outline logs deploy/outline -c outline | grep -E 'OIDC endpoints mounted|Running migrations'
kubectl get databases.postgresql.cnpg.io -n postgres postgres-cluster-outline
```

`OIDC endpoints mounted after discovery` means the pod reached tsidp. Then open
`https://outline.tail19032f.ts.net` from a tailnet device.

If the hostname does not come up, check the Tailscale `autoApprovers` first
([lessons-learned.md](lessons-learned.md), the Trilium entry): the ProxyGroup
replica that advertises the new VIP Service is tagged `tag:k8s`.

## Login

The login page shows one button, **Continue with Tailscale**. It goes to
tsidp, which signs in whoever reaches it over the tailnet, and returns to
Outline. The first user to sign in creates the workspace and becomes its
admin.

- **Identity.** Outline matches a returning user on the OIDC `sub` claim
  (tsidp's `userid:<number>`), not on the email. The email is the Tailscale
  login name (the Google address for the owner), as described in
  [tsidp.md](tsidp.md#claims).
- **Logout** works because `OIDC_DISABLE_REDIRECT=true`. tsidp has no
  `end_session_endpoint`, so logging out of Outline cannot end the tsidp
  session. Without this setting the login page would send the browser straight
  back to tsidp, which would sign it in again at once.

**Who can sign in.** Any tailnet identity that reaches tsidp gets a token, and
Outline then creates an account for it. Outline's own restrictions do not help
here. "Require invites" exists only in Outline's cloud-hosted edition. The
other two, an invitation and the workspace's allowed-domains list, accept a
login only when the provider marks the email as verified. tsidp sends no
`email_verified` claim, so an invited person could not sign in, and once any
allowed domain is set, no new user could (`server/commands/userProvisioner.ts`,
as of v1.11.0). Leave the allowed-domains list empty. The boundary is
therefore the tailnet policy ([tsidp.md](tsidp.md#tailnet-policy)). With
today's allow-all rule, every tailnet member can create an Outline account.
Revisit this before another person joins the tailnet.

## Connecting Claude Code over MCP

Only clients running on a tailnet device can use the endpoint. Claude's custom
connectors (claude.ai, Claude Desktop, Cowork, mobile) connect from
Anthropic's cloud and cannot reach a tailnet-only host (#376, Background).

MCP is on by default for a workspace. The switch is **Settings → Workspace →
AI → MCP server**.

**Authentication: OAuth.** Outline's MCP endpoint accepts OAuth or an API key
sent as a bearer token. OAuth is used here: Claude Code registers itself with
Outline (dynamic client registration), the owner approves it in the browser,
and Claude Code keeps and refreshes the token itself. An API key passed with
`--header` would instead be written into Claude Code's MCP configuration, and
it does not expire unless one is set when it is created. Use an API key only for a client
that cannot do OAuth.

On the Mac, once, at user scope so every project sees it:

```bash
claude mcp add --transport http --scope user outline https://outline.tail19032f.ts.net/mcp
```

Then, in a Claude Code session, run `/mcp`, choose `outline`, and complete the
login in the browser (signing in to Outline via tsidp if needed). The tools
appear as `mcp__outline__*`.

To revoke access, revoke Claude Code under **Settings → Account → API &
Access → Application access**, and remove the server on the Mac with
`claude mcp remove outline --scope user`.

### Check the MCP connection

This is the go/no-go gate of #378. From a Claude Code session with the server
connected, each of these must succeed and be visible in the Outline UI:

| Operation | MCP tool |
|-----------|----------|
| Search finds an existing document | `list_documents` with `query` |
| A document's content is read | `fetch` |
| A new document is created in a collection | `create_document` |
| An existing document is edited | `update_document` |

Re-run it after an Outline upgrade: the Renovate PR for `outlinewiki/outline`
links here.

## Backup and Restore

| Data | Where | Backed up by |
|------|-------|--------------|
| Documents, users, settings | `outline` database | CloudNativePG barman backup to R2, daily 18:00 UTC ([cloudnative-pg.md](cloudnative-pg.md)) |
| Attachments and images | PVC `outline-data` | K8up, daily 19:00 UTC, into `homelab-k8up-backups/outline` ([k8up.md](k8up.md)) |
| Redis | in-pod, not persisted | nothing; it holds queues and caches only |

The two halves are restored together and must be paired: never pair a
database backup that is newer than the volume backup, or documents can
reference attachments the restored volume does not have
([cloudnative-pg.md](cloudnative-pg.md#backup-configuration-notes)).

- Database: [cloudnative-pg.md — Restore from Backup](cloudnative-pg.md#restore-from-backup).
  The cluster is shared, so a restore goes through a separate recovery cluster;
  copy the `outline` database from it.
- Attachments: [k8up.md — Restore](k8up.md#restore).

### Check that attachments survive a restart

```bash
# 1. Upload an image to any document in the UI, then confirm the file exists
kubectl -n outline exec deploy/outline -c outline -- find /var/lib/outline/data -type f | head
# 2. Restart the pod
kubectl -n outline delete pod -l app=outline
kubectl -n outline wait --for=condition=Ready pod -l app=outline --timeout=5m
# 3. Reload the document: the image must still display
```

## Upgrades

Renovate proposes new `outlinewiki/outline` tags. Outline runs its database
migrations on startup, one way, so the PR body asks for a database and volume
backup before merging. A revert restores the image but not the old schema.

## Troubleshooting

### Outline does not become Ready

Read the current log (`kubectl -n outline logs deploy/outline -c outline`).
Outline's master process supervises a forked web process and forks it again
when it exits. A failure inside that process therefore shows up as a repeating
error in the current log, while the container keeps running.

- `Failed to discover OIDC configuration`: the pod cannot reach tsidp. The web
  process exits and is forked again, so the error repeats and readiness flaps.
  The liveness probe may never restart the container, because `/_health` can
  answer between re-forks, so delete the pod once the cause is fixed. Check
  from a throwaway pod:

  ```bash
  kubectl run tsidp-check --rm -i --restart=Never --image=curlimages/curl:latest --command -- \
    curl -sS -o /dev/null -w '%{http_code}\n' --max-time 15 \
    --resolve idp.tail19032f.ts.net:443:100.89.177.82 \
    https://idp.tail19032f.ts.net/.well-known/openid-configuration
  ```

  Expect `200`. If it fails, check that tsidp is up ([tsidp.md](tsidp.md)),
  that its IP is still the one in `hostAliases` (`tailscale ip -4 idp`), and
  that the node is online in the tailnet.
- Database errors at startup stop the container itself, because the
  migration check runs from the master process. Read the previous log with
  `--previous`:
  - `Could not connect to the database`: the host or Service is unreachable.
    Check the cluster (`kubectl -n postgres get clusters.postgresql.cnpg.io`).
  - `password authentication failed for user "outline"`: the password in
    `outline-db` does not match `postgres-outline-user`, or the role does not
    exist yet (`kubectl -n postgres get databases.postgresql.cnpg.io`).

### Login fails

- tsidp shows an error instead of returning to Outline: the client ID in the
  `outline` Secret is unknown to tsidp, or the client's redirect URI is not
  exactly `https://outline.tail19032f.ts.net/auth/oidc.callback`.
- tsidp returns to Outline, which then reports an authentication error, and the
  log shows a 401 or `invalid_client` from the token endpoint: the client
  secret does not match.

## References

- [Outline hosting docs](https://docs.getoutline.com/s/hosting) — environment, OIDC, file storage
- [Outline MCP guide](https://docs.getoutline.com/s/guide/doc/mcp-6j9jtENNKL)
- [Claude Code: connect to MCP servers](https://code.claude.com/docs/en/mcp)
- [tsidp.md](tsidp.md)
