# tsidp - Tailnet OIDC Provider

[tsidp](https://github.com/tailscale/tsidp) is an OpenID Connect / OAuth
identity provider that signs users in with their Tailscale identity. It gives
self-hosted apps that need an external IdP a login without running a separate
account store. Introduced by
[#377](https://github.com/aoshimash/homelab-k8s/issues/377) as the login for
Outline ([#376](https://github.com/aoshimash/homelab-k8s/issues/376)), which
has no built-in password login.

**tsidp is experimental.** Its README marks it as a community project under
active development that may ship breaking changes, and requires
`TAILSCALE_USE_WIP_CODE=1` while tsidp is below v1.0.0. Read the release notes
before merging a Renovate bump.

## Current Configuration

| Item | Value |
|------|-------|
| Manifests | `k8s/configs/tsidp/` (the `configs` layer, so it is up before `apps`) |
| Image | `ghcr.io/tailscale/tsidp`, pinned `tag@sha256` in `deployment.yaml` |
| URL | `https://idp.<tailnet>.ts.net` (`TS_HOSTNAME=idp`) |
| Tailnet identity | Its own node, joined via tsnet, tagged `tag:tsidp` |
| State | `/data` on PVC `tsidp-data` (local-path), K8up-backed |
| Pod user | 1001 (the image's `app` user), read-only root filesystem |

tsidp joins the tailnet itself and listens only there. It has no Service,
Ingress, or container port, and it does not use the Tailscale operator.

### What the state volume holds

`/data` holds the tsnet node state (the node's tailnet identity), the OIDC
signing key, and the registered OIDC clients with their secrets. Losing it
re-registers tsidp as a new node with a new signing key and drops every client
registration. Every app's client ID and secret then stop working. That is why
the volume is backed up (see [k8up.md](k8up.md)) and the Deployment uses
`Recreate`: two pods on one state directory would be the same node twice.

### No probes, and why `-local-port` stays off

Because tsidp listens only on the tailnet, the kubelet has nothing to probe.
tsidp's `-local-port` flag would add a localhost listener, but tsidp treats
every loopback request as fully privileged: it gets admin-UI and
dynamic-client-registration rights without any tailnet identity (the
loopback check in `server/appcap.go`, as of v0.0.17). Do not enable it to get
a probe target.

## Tailnet Policy

The tailnet policy file is edited in the
[admin console](https://login.tailscale.com/admin/acls), not in Git, so this
section is the record of what tsidp needs there. Keep it in sync when the
policy changes.

```hujson
{
  "tagOwners": {
    // Lets an admin authenticate the tsidp node with this tag on first start.
    "tag:tsidp": ["autogroup:admin"],
  },
  "grants": [
    // Who can sign in: tsidp issues a token to any tailnet identity that
    // reaches it on 443 (see below), so this grant is the login boundary.
    {
      "src": ["autogroup:admin"],
      "dst": ["tag:tsidp"],
      "ip": ["tcp:443"],
    },
    // Admin UI, used to register OIDC clients. Dynamic client registration
    // (allow_dcr) is not granted: clients are registered by hand.
    {
      "src": ["autogroup:admin"],
      "dst": ["tag:tsidp"],
      "app": {
        "tailscale.com/cap/tsidp": [{ "allow_admin_ui": true }],
      },
    },
  ],
}
```

Both the admin UI and dynamic client registration are denied unless a grant
allows them. Capability grants take effect immediately, with no restart.

**Network access is the login boundary.** Signing in needs no capability:
tsidp issues a token to whichever user identity reaches its `/authorize`
endpoint over HTTPS (tagged devices are refused). So who can log in to an app
behind tsidp is decided by who can reach `tag:tsidp` on TCP 443, and the first
grant above limits that to tailnet admins — the owner.

That limit only holds if nothing broader also reaches `tag:tsidp`. Tailscale
initializes a tailnet with an allow-all policy (any source to any destination),
and access rules are additive, so with that rule in place every tailnet member can sign
in. When applying the policy, check for such a rule. Remove or narrow it, or,
if keeping it is a deliberate choice (e.g. the owner is the only member),
record that here.

A relying party that calls the token and userinfo endpoints from its own
server needs reach to `tag:tsidp:443` as well. Add it with the relying party
that needs it.

## First Start

No auth key is stored anywhere. On its first start, with an empty `/data`,
tsidp prints a login URL to its log. Authenticating it once registers the node
with `tag:tsidp` (from `TS_ADVERTISE_TAGS`), and the state on `/data` keeps it
registered from then on.

1. Add `tag:tsidp` to `tagOwners` (see [Tailnet Policy](#tailnet-policy)).
   Authentication with the tag fails until it exists there.
2. After Flux reconciles, read the login URL:

   ```bash
   kubectl -n tsidp logs deploy/tsidp | grep -i login
   ```

3. Open the URL and authenticate as a tailnet admin.
4. In the admin console, check that the `idp` machine shows the `tag:tsidp`
   tag and **key expiry disabled**. Tailscale disables key expiry by default
   for a device that is tagged when it first authenticates. If expiry shows as
   enabled, disable it on that machine. Otherwise the node drops off the
   tailnet when its key expires, and every login breaks with it.
5. Confirm the discovery document loads from a tailnet device. The first
   request can take a few minutes while the TLS certificate is issued:

   ```bash
   curl -fsS https://idp.<tailnet>.ts.net/.well-known/openid-configuration
   ```

### Check that the state survives a restart

Do this after at least one client exists ([Registering an OIDC
Client](#registering-an-oidc-client)), so the check covers clients as well as
the node identity.

```bash
kubectl -n tsidp delete pod -l app=tsidp
kubectl -n tsidp wait --for=condition=Ready pod -l app=tsidp --timeout=2m
# The new pod logs this only after it has joined the tailnet
kubectl -n tsidp logs deploy/tsidp | grep 'tsidp server started'
# Expect no output: a login URL here means the node identity was lost
kubectl -n tsidp logs deploy/tsidp | grep -i 'login.tailscale.com'
```

Then check:

- The admin console still shows a single `idp` machine. A second one
  (`idp-1`) means the state was not persisted.
- The admin UI at `https://idp.<tailnet>.ts.net/` still lists the client,
  with the same client ID.

## Registering an OIDC Client

1. Open `https://idp.<tailnet>.ts.net/` as a user with the `allow_admin_ui`
   grant.
2. Create a client with the app's redirect URI(s). tsidp shows the client ID
   and a generated secret.
3. Store both in the app's SOPS-encrypted secret, and point the app's issuer
   at `https://idp.<tailnet>.ts.net`. Its endpoints are listed in the discovery
   document above.

### Claims

The `email` claim in the ID token and in `/userinfo` is the user's Tailscale
login name, e.g. the Google address for a Google login. GitHub and passkey
logins have login names that are not real addresses (`user@github`,
`user@passkey`); for those tsidp appends its own DNS name, giving e.g.
`user@github.idp.<tailnet>.ts.net` (`realishEmail` in `server/server.go`, as
of v0.0.17). Apps
that key users on email, such as Outline, see exactly this value.

Extra claims can be added per grant with `extraClaims` (and exposed in
`/userinfo` with `includeInUserInfo`). None are configured.

### Check the claims for a user

To see what an app will receive before configuring one, run the
authorization-code flow by hand from a tailnet device with a throwaway client:

1. In the admin UI, create a client named `claims-check` with redirect URI
   `http://localhost:8765/callback`. Nothing has to listen on that port.
2. Open this URL in a browser on a tailnet device, signed in to Tailscale as
   the user to check:

   ```
   https://idp.<tailnet>.ts.net/authorize?response_type=code&client_id=<CLIENT_ID>&redirect_uri=http://localhost:8765/callback&scope=openid%20email%20profile&state=x
   ```

   The browser is redirected to `localhost` and shows a connection error. Copy
   the `code` parameter from the address bar.
3. Exchange the code and read `/userinfo`:

   ```bash
   IDP=https://idp.<tailnet>.ts.net
   TOKEN=$(curl -fsS -u "<CLIENT_ID>:<CLIENT_SECRET>" "$IDP/token" \
     -d grant_type=authorization_code -d code=<CODE> \
     -d redirect_uri=http://localhost:8765/callback | jq -r .access_token)
   curl -fsS -H "Authorization: Bearer $TOKEN" "$IDP/userinfo" | jq
   ```

4. Note the `email` value an app will see (record it in the issue or PR that
   configures the app), then delete the `claims-check` client in the admin UI.

## Backup and Recovery

`tsidp-data` is backed up daily by K8up into `homelab-k8up-backups/tsidp`
(see [k8up.md](k8up.md)). Restoring it brings back the same node, signing key,
and clients, so no app needs reconfiguring.

If the state is lost and cannot be restored:

1. Delete the old `idp` machine in the admin console. Otherwise the new
   registration comes up as `idp-1`, at a different URL.
2. Restart tsidp and authenticate the new login URL ([First Start](#first-start)).
3. Re-register every client and update each app's secret. The client IDs,
   secrets, and signing key are all new.

## Not Yet Covered

- **Pods reaching tsidp.** Cluster workloads are not on the tailnet and cannot
  resolve `*.ts.net` names, so an app running in the cluster cannot reach
  `https://idp.<tailnet>.ts.net` by default. The first such app is Outline;
  giving it a path to the tailnet is part of
  [#378](https://github.com/aoshimash/homelab-k8s/issues/378).

## References

- [tsidp README](https://github.com/tailscale/tsidp)
- [Tailscale: grants and app capabilities](https://tailscale.com/kb/1537/grants-app-capabilities)
- [Tailscale: key expiry](https://tailscale.com/kb/1028/key-expiry)
- [Tailscale: tags](https://tailscale.com/kb/1068/tags)
