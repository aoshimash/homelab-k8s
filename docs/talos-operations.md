# Talos Linux Day 2 Operations

This document describes common Day 2 operations for the Talos Linux cluster.

## Prerequisites

Ensure you have the following environment variables set:

```bash
export TALOSCONFIG=/path/to/infra/talos/clusterconfig/talosconfig
export SOPS_AGE_KEY_FILE=/path/to/age.agekey
```

## Upgrading Talos Linux

### When to Upgrade

- **Talos version upgrade**: New Talos Linux version available
- **Extension changes**: Adding/removing system extensions (e.g., iscsi-tools, tailscale)
- **Configuration changes**: Modifying `talconfig.yaml` settings

### Upgrade Procedure

#### 1. Update talconfig.yaml

Edit `infra/talos/talconfig.yaml` with the new configuration:

```yaml
nodes:
  - hostname: homelab-node-01
    # Update the Image Factory URL with new schematic
    talosImageURL: factory.talos.dev/installer/<new-schematic-id>
```

#### 2. Create New Image Schematic (if needed)

If adding/removing extensions, create a new schematic at [Talos Image Factory](https://factory.talos.dev/):

1. Select your architecture (amd64)
2. Select platform (metal)
3. Select Talos version
4. Add required extensions
5. Copy the generated Schematic ID

**Current Extensions:**
- `siderolabs/iscsi-tools` - Required for Longhorn storage
- `siderolabs/tailscale` - VPN connectivity

#### 3. Perform the Upgrade

> ⚠️ **Single Node Cluster Warning**: By default, `talosctl upgrade` stops all pods
> before upgrading, which can cause issues on single-node clusters. The control plane
> components (kube-apiserver, etcd) will also be stopped, potentially causing the
> upgrade to hang waiting for kubelet lifecycle finalizers.
> See [GitHub Issue #11775](https://github.com/siderolabs/talos/issues/11775) for details.

**For single-node clusters, always use the `--preserve` flag:**

```bash
# Upgrade the node with new image (single-node cluster)
talosctl upgrade --nodes 192.168.0.10 \
  --image factory.talos.dev/installer/<schematic-id>:v1.12.0 \
  --preserve

# Monitor the upgrade progress
talosctl dmesg -f --nodes 192.168.0.10
```

The `--preserve` flag:
- Skips the cordon/drain process
- Does not stop pods before upgrade
- Allows the upgrade to proceed without waiting for pod termination

The node will:
1. Download the new image
2. Stage the upgrade
3. Reboot with the new image

#### 4. Verify the Upgrade

```bash
# Check Talos version
talosctl version --nodes 192.168.0.10

# Verify extensions are loaded
talosctl get extensions --nodes 192.168.0.10

# Check node health
talosctl health --nodes 192.168.0.10

# Verify Kubernetes node status
kubectl get nodes
```

### Rollback Procedure

If the upgrade fails, Talos automatically boots from the previous image. To manually rollback:

```bash
talosctl rollback --nodes 192.168.0.10
```

## Renovate upgrade flow

Renovate opens PRs to bump the Talos OS image and Kubernetes versions referenced
in `infra/talos/talconfig.yaml`. These PRs do **not** apply the upgrade to the
cluster — they only update the desired state in Git. The operator must run the
relevant `task` targets manually after merge.

### PR review criteria

Before merging a Renovate PR, verify:

- **CHANGELOG / release notes**: Read the upstream Talos and Kubernetes release
  notes for the proposed version. Look for breaking changes, deprecated flags,
  and new minimum requirements.
- **Cilium compatibility**: Confirm the new Kubernetes version is supported by
  the currently deployed Cilium release. Cross-check the Cilium support matrix
  before approving Kubernetes minor-version bumps.
- **Breaking-change scan**: Search the diff for changes to `talconfig.yaml`
  schema, machine config keys, or extension references that imply a manual
  follow-up step.

### Manual operation after merge

Once the PR is merged on `main`, apply the change to the cluster promptly:

```bash
git pull
task talos:upgrade        # for Talos OS
task talos:upgrade-k8s    # for Kubernetes
```

> **Important**: Apply promptly after merge to prevent drift between Git and
> cluster state. The longer the desired state in Git diverges from the running
> cluster, the harder it is to reason about subsequent PRs.

### Verification

```bash
task talos:health
kubectl get nodes
```

Confirm the node is `Ready`, the reported Talos and Kubernetes versions match
the merged PR, and all critical workloads return to a healthy state.

## Rollback strategy

This is a **single-node cluster**, so an upgrade failure brings the entire
cluster down. Always plan for rollback before applying a Renovate PR.

### Take an etcd snapshot before upgrading

```bash
talosctl etcd snapshot db.snapshot
```

Store the snapshot off-node. If the upgrade corrupts cluster state beyond what
A/B partition rollback can recover, the snapshot is the last line of defense.

### Immediate rollback (within ~30 min of upgrade)

Talos preserves the previous install image in the secondary A/B partition. If
the new image fails to boot or the node is misbehaving immediately after
upgrade, revert to the previous partition:

```bash
talosctl rollback -n 192.168.0.10
```

This is the fastest and safest rollback path. It does **not** require any Git
changes — the running image simply switches back.

### Late rollback (after the previous partition is gone)

Once another upgrade has overwritten the secondary partition, `talosctl
rollback` can no longer recover the prior version. Roll back via Git instead:

1. Revert the merged Renovate PR:
   ```bash
   git revert <merge-commit>
   # or, if the PR is still open
   gh pr revert <pr-number>
   ```
2. Re-run the upgrade against the now-restored older version:
   ```bash
   task talos:upgrade
   ```
3. Reboot manually if the node does not pick up the change automatically:
   ```bash
   talosctl reboot --nodes 192.168.0.10
   ```

### Single-node warning

A failed upgrade on this cluster equals **full cluster downtime** — there is no
peer node to keep workloads running. Always:

- Take an etcd snapshot (`talosctl etcd snapshot db.snapshot`) before applying
  a Renovate upgrade PR.
- Schedule upgrades during a maintenance window when downtime is acceptable.
- Keep a console / out-of-band path to the node so a stuck boot can be
  diagnosed without the network.

## Future automation options

The current flow is intentionally manual: Renovate proposes the version bump
in Git, an operator reviews and applies it. The options below are recorded for
future consideration but are **not** implemented today.

- **tuppr controller (multi-node)**: CRD-based rolling Talos upgrades driven
  by an in-cluster controller. Reconsider when the cluster grows beyond a
  single node — rolling upgrades have no benefit on a single-node cluster and
  add operational surface area.
- **Drift detection automation**: A GitHub Actions cron job that compares the
  desired Talos / Kubernetes version in Git against the live cluster and files
  an issue when drift is detected. Estimated setup: ~1–2 hours. Useful as a
  safety net for the "apply promptly after merge" rule above.
- **Full automation**: A `workflow_dispatch` button (or tuppr) that applies
  Renovate-merged upgrades automatically. Considered too risky for a
  single-node cluster — keep the human gate until there is a multi-node
  failure domain.

## Adding System Extensions

### Common Extensions

| Extension | Purpose | Required For |
|-----------|---------|--------------|
| `siderolabs/iscsi-tools` | iSCSI initiator | Longhorn storage |
| `siderolabs/tailscale` | VPN client | Remote access |
| `siderolabs/util-linux-tools` | Misc utilities | Various tools |

### Procedure

1. Go to [Talos Image Factory](https://factory.talos.dev/)
2. Select current extensions + new extension
3. Copy the new Schematic ID
4. Update `talconfig.yaml` with new `talosImageURL`
5. Follow the [Upgrade Procedure](#upgrade-procedure)

## Applying Configuration Changes

For configuration changes that don't require a new image:

```bash
cd infra/talos

# Regenerate configuration
talhelper genconfig --no-gitignore

# Apply configuration (may require reboot)
talosctl apply-config \
  --nodes 192.168.0.10 \
  --file clusterconfig/homelab-cluster-homelab-node-01.yaml

# Check if reboot is required
talosctl get machinestatus --nodes 192.168.0.10
```

## Rotating the Tailscale Auth Key (Talos)

Tailscale auth keys (`tskey-auth-...`) **cannot be retrieved after creation**.
If the key is lost, revoked, expired, or the node needs to re-authenticate, you must generate a new key and update Talos.

### 1. Create a new Auth Key in the Tailscale Admin Console

Use an auth key appropriate for a long-lived node:
- Prefer **Reusable: ON** and **Ephemeral: OFF**
- Consider **Pre-approved: ON** to avoid manual approval during bootstrap

### 2. Update the SOPS-managed environment file

Update `infra/talos/talenv.sops.yaml` (encrypted) with the new key.

> Tip: Prefer updating via SOPS so plaintext doesn't accidentally land in Git.

### 3. Regenerate and apply Talos configuration

```bash
cd infra/talos

# Regenerate configs (requires SOPS_AGE_KEY_FILE)
talhelper genconfig

# Apply to the node
talosctl apply-config --nodes 192.168.0.10 \
  --file clusterconfig/homelab-cluster-homelab-node-01.yaml
```

### 4. Verify Tailscale is logged in and stable

```bash
# Confirm the tailscale extension is loaded
talosctl get extensions --nodes 192.168.0.10

# Confirm the auth key is injected into the extension service config
talosctl get extensionserviceconfigs --nodes 192.168.0.10 -o yaml

# Restart tailscale extension service (optional, but speeds up troubleshooting)
talosctl service ext-tailscale restart --nodes 192.168.0.10

# Check tailscale logs for "Running" state and assigned 100.x address
talosctl logs ext-tailscale --nodes 192.168.0.10
```

## Image Factory Reference

### Current Schematic

- **Schematic ID**: `e2e3b54334c85fdef4d78e88f880d185e0ce0ba0c9b5861bb5daa1cd6574db9b`
- **Talos Version**: v1.12.0
- **Extensions**:
  - `siderolabs/iscsi-tools`
  - `siderolabs/tailscale`
- **Image Factory URL**: [View/Edit Schematic](https://factory.talos.dev/?arch=amd64&bootloader=auto&cmdline-set=true&extensions=-&extensions=siderolabs%2Fiscsi-tools&extensions=siderolabs%2Ftailscale&platform=metal&target=metal&version=1.12.0)

### Image URLs

| Type | URL |
|------|-----|
| Installer | `factory.talos.dev/installer/e2e3b54334c85fdef4d78e88f880d185e0ce0ba0c9b5861bb5daa1cd6574db9b:v1.12.0` |
| ISO | `https://factory.talos.dev/image/e2e3b54334c85fdef4d78e88f880d185e0ce0ba0c9b5861bb5daa1cd6574db9b/v1.12.0/metal-amd64.iso` |

## Troubleshooting

### Extension Not Working

```bash
# Verify extension is loaded
talosctl get extensions --nodes 192.168.0.10

# Check extension services
talosctl services --nodes 192.168.0.10
```

### Upgrade Stuck

```bash
# Check upgrade status
talosctl upgrade --nodes 192.168.0.10 --stage

# Force reboot if needed
talosctl reboot --nodes 192.168.0.10
```

### Node Not Coming Back After Upgrade

1. Check physical console for errors
2. Verify network connectivity
3. Try rollback: `talosctl rollback --nodes 192.168.0.10`
