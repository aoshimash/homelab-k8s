# Talos Linux Day 2 Operations

This document describes common Day 2 operations for the Talos Linux cluster.

## Prerequisites

Ensure you have the following environment variables set:

```bash
export TALOSCONFIG=/path/to/infra/talos/clusterconfig/talosconfig
export SOPS_AGE_KEY_FILE=/path/to/age.agekey
```

> **Reaching the node — LAN vs Tailscale**: The commands below use the node's
> LAN address `192.168.0.10` (the value baked into `talconfig.yaml` and the
> generated `talosconfig`). That only works from the home LAN. When operating
> from outside the LAN (i.e. over the tailnet — the common case for a laptop),
> the LAN address is unreachable (`dial tcp 192.168.0.10:50000: connect: no
> route to host`). Override the endpoint and node with the node's **tailnet IP**
> instead, e.g. `--endpoints <tailnet-ip> --nodes <tailnet-ip>`. Find it with
> `kubectl get node -o wide` (INTERNAL-IP) or `tailscale status`.

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

> ⚠️ **Single Node Cluster Warning**: By default, `talosctl upgrade` cordons the
> node and drains its pods before rebooting. On a single-node cluster the drain
> cannot succeed — there is nowhere to move the pods, and any PodDisruptionBudget
> with `ALLOWED DISRUPTIONS = 0` (Longhorn `instance-manager`, the CloudNativePG
> primary) blocks eviction until the drain times out. The upgrade can also hang
> waiting for kubelet lifecycle finalizers.
> See [GitHub Issue #11775](https://github.com/siderolabs/talos/issues/11775) for details.

**For single-node clusters, skip the drain with `--drain=false`:**

```bash
# Upgrade the node with new image (single-node cluster)
talosctl upgrade --nodes 192.168.0.10 \
  --image factory.talos.dev/installer/<schematic-id>:v1.13.4 \
  --drain=false

# Monitor the upgrade progress
talosctl dmesg -f --nodes 192.168.0.10
```

`--drain=false`:
- Skips the cordon/drain process entirely, so the upgrade does not stall on the
  un-satisfiable single-node drain or on blocking PodDisruptionBudgets.
- Is not a "hard kill": Talos still runs its graceful shutdown sequence, sending
  SIGTERM + grace period to pods during the reboot. Stateful workloads
  (CloudNativePG, Longhorn) recover via normal crash recovery on restart.

> **Note — `--preserve` is gone:** Older docs and Talos guides told single-node
> operators to pass `--preserve`. That flag controlled *ephemeral-partition data
> preservation* (not the drain), and it was **removed in `talosctl` 1.13.0+** —
> passing it now errors with `unknown flag: --preserve`. Data preservation is the
> default in 1.13+, and the single-node concern (skipping the drain) is handled by
> `--drain=false`. Do not add `--preserve`.

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
cluster — they only update the desired state in Git. The operator must apply
the change manually with `talosctl` after merge.

Talos OS and Kubernetes upgrades are kept as **separate Renovate PRs** because
their apply procedures and blast radius differ:

- A Talos OS bump (`talosVersion`) reboots the node — full control plane is
  unavailable for the duration of the reboot.
- A Kubernetes bump (`kubernetesVersion`) updates control-plane static pods
  and kubelet — the apiserver bounces briefly but the node stays up.

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

### Manual operation after merge — Talos OS bump

Once a `talosVersion` Renovate PR is merged on `main`:

```bash
git pull
cd infra/talos

# Regenerate clusterconfig from the updated talconfig.yaml
SOPS_AGE_KEY_FILE=../../age.agekey talhelper genconfig

# Build the full image URL (talosImageURL holds only the schematic; the version
# tag comes from talosVersion).
TALOS_IMAGE=$(yq '.nodes[0].talosImageURL' talconfig.yaml)
TALOS_VERSION=$(yq '.talosVersion' talconfig.yaml)

talosctl --talosconfig clusterconfig/talosconfig \
  --endpoints 192.168.0.10 --nodes 192.168.0.10 \
  upgrade --image "${TALOS_IMAGE}:${TALOS_VERSION}" --drain=false --wait
```

> Operating over the tailnet? Replace `192.168.0.10` with the node's tailnet IP
> in both `--endpoints` and `--nodes` (see the note under
> [Prerequisites](#prerequisites)).

After `talosctl` reports `post check passed`, the kube-apiserver still needs a
moment to come back up. Block until ready before running anything else against
the cluster:

```bash
task talos:health      # blocks until Talos and K8s control plane are ready
kubectl get nodes      # node should show the new Talos version
```

#### Verify all workloads recovered

A Talos OS upgrade reboots the node, which means every Cilium agent restarts
and re-attaches its eBPF programs to the new kernel. On a kernel-version bump
(e.g. Talos 1.12 → 1.13) this re-attach can land in an inconsistent state
that breaks pod-to-external SNAT or kubelet → pod liveness probes, even
though `task talos:health` shows the cluster as healthy. Always sweep for
unhealthy pods before walking away:

```bash
kubectl get pods -A | grep -vE 'Running|Completed'
```

After a reboot this is usually **not** empty, and that is normal: the reboot
(especially with `--drain=false`) leaves `Failed`-phase residue — orphaned
Deployment replicas whose Deployment is already back to its full ready count,
plus `Job` pods that fired and failed during the recovery window. These are
harmless leftovers, not an outage. Confirm recovery by the things that actually
matter, not by an empty sweep:

```bash
kubectl get deploy,ds -A     # READY should read N/N for every controller
kubectl get pvc -A | grep -v Bound                       # all volumes Bound? (empty = good)
kubectl get pods -A | grep -E 'audiobookshelf|home-assistant|vikunja'  # user apps Running?
```

Once those are healthy, review the residue before clearing it — a blanket
`--field-selector=status.phase=Failed` sweep also deletes anything that failed
moments ago for an unrelated reason, destroying the only evidence (logs, exit
code) before anyone looked at it. Check each pod's age first:

```bash
kubectl get pods -A --field-selector=status.phase=Failed \
  -o custom-columns='NAMESPACE:.metadata.namespace,NAME:.metadata.name,CREATED:.metadata.creationTimestamp'
```

Pods whose `CREATED` timestamp lines up with this reboot are safe to delete —
either by name, or in bulk once you've confirmed nothing unrelated is mixed in:

```bash
kubectl delete pods -A --field-selector=status.phase=Failed
```

If this step is skipped, it is not silent forever: the `FailedPodResidue`
alert (`k8s/infrastructure/grafana-alloy/prometheusrule-failed-pods.yaml`)
fires to `#homelab-alerts` once a pod has sat in `Failed` phase for over 6h,
so a forgotten cleanup still surfaces on its own. See
[#246](https://github.com/aoshimash/homelab-k8s/issues/246) — the alert is
intentionally notify-only; a human decides whether to delete, not a
scheduled job.

A pod is a **real** failure only if it stays in `CrashLoopBackOff`,
`Init:CrashLoopBackOff`, or stuck `0/N Ready` while its controller still wants
it (the ready count above never reaches full). In that case:

1. Restart the Cilium DaemonSet so the agent re-attaches eBPF cleanly:
   ```bash
   kubectl -n kube-system rollout restart ds/cilium
   kubectl -n kube-system rollout status ds/cilium
   ```
2. If pods are still flapping after Cilium recovers, check the agent log
   for backend incompatibilities introduced by the new kernel:
   ```bash
   kubectl -n kube-system logs ds/cilium --tail=100 | grep -iE 'error|fail|incompat'
   ```
   On Talos 1.13+, the agent logs `iptables ... (nf_tables): table 'nat'
   is incompatible` every 10 seconds. This is **cosmetic noise** — the
   kernel switched its iptables backend to nftables but Cilium 1.19.x still
   probes the legacy table. Pod-to-external SNAT runs in eBPF via
   `bpf.masquerade=true` and is not affected. The Helm value
   `installIptablesRules: false` silently has no effect in Cilium 1.19.x, so the
   noise is tolerated until upstream Cilium ships a real nf_tables-only
   mode. Treat the line as expected and grep around it when investigating
   real failures.
3. If kubelet liveness probes are timing out for specific Flux / app pods
   under stricter Cilium NetworkPolicy enforcement, confirm no NetworkPolicy
   selects the pod with `from` clauses that exclude the host network
   (`namespaceSelector: {}` or `podSelector: {}` does not match host probes
   in Cilium 1.19+).

> Note: `--drain=false` is required on this single-node cluster — the default
> drain cannot succeed and stalls on blocking PodDisruptionBudgets. Do **not**
> pass the legacy `--preserve` flag; it was removed in `talosctl` 1.13.0+ and now
> errors out. See [Perform the Upgrade](#3-perform-the-upgrade) for the rationale.

### Manual operation after merge — Kubernetes bump

Once a `kubernetesVersion` Renovate PR is merged on `main`:

```bash
git pull
cd infra/talos

K8S_VERSION=$(yq '.kubernetesVersion' talconfig.yaml)

talosctl --talosconfig clusterconfig/talosconfig \
  --endpoints 192.168.0.10 --nodes 192.168.0.10 \
  upgrade-k8s --to "${K8S_VERSION}"
```

Verification:

```bash
task talos:health
kubectl get nodes      # node version reflects the new K8s release
kubectl -n kube-system get pods   # control-plane pods all Running
```

> **Important**: Apply promptly after merge to prevent drift between Git and
> cluster state. The longer the desired state in Git diverges from the running
> cluster, the harder it is to reason about subsequent PRs.

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
talosctl rollback --nodes 192.168.0.10
```

This is the fastest and safest rollback path. It does **not** require any Git
changes — the running image simply switches back.

### Late rollback (after the previous partition is gone)

Once another upgrade has overwritten the secondary partition, `talosctl
rollback` can no longer recover the prior version. Roll back via Git instead:

1. Revert the merged Renovate PR:
   ```bash
   git revert <merge-commit>
   # or, after the PR was merged, open a revert PR:
   gh pr revert <pr-number>
   ```
2. Re-run the upgrade against the now-restored older version. The
   command is documented in the
   [Manual operation after merge — Talos OS bump](#manual-operation-after-merge--talos-os-bump)
   section above (`talosctl ... upgrade --image ...`).
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
- **Talos Version**: v1.13.4
- **Extensions**:
  - `siderolabs/iscsi-tools`
  - `siderolabs/tailscale`
- **Image Factory URL**: [View/Edit Schematic](https://factory.talos.dev/?arch=amd64&bootloader=auto&cmdline-set=true&extensions=-&extensions=siderolabs%2Fiscsi-tools&extensions=siderolabs%2Ftailscale&platform=metal&target=metal&version=1.13.4)

### Image URLs

| Type | URL |
|------|-----|
| Installer | `factory.talos.dev/installer/e2e3b54334c85fdef4d78e88f880d185e0ce0ba0c9b5861bb5daa1cd6574db9b:v1.13.4` |
| ISO | `https://factory.talos.dev/image/e2e3b54334c85fdef4d78e88f880d185e0ce0ba0c9b5861bb5daa1cd6574db9b/v1.13.4/metal-amd64.iso` |

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
