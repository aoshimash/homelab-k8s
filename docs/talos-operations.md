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

#### 3. Regenerate Talos Configuration

```bash
cd infra/talos

# Regenerate configs with updated settings
talhelper genconfig --no-gitignore
```

#### 4. Perform the Upgrade

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

#### 5. Verify the Upgrade

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
