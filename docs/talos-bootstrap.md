# Talos Linux Bootstrap Guide

This document describes the initial bootstrap process for the homelab Kubernetes cluster using Talos Linux.

## Prerequisites

### Required Tools

Install the following tools:

```bash
# Talos CLI
brew install siderolabs/tap/talosctl

# Talhelper (Talos configuration helper)
brew install budimanjojo/tap/talhelper

# SOPS (Secrets management)
brew install sops

# Age (Encryption)
brew install age
```

### Network Requirements

- Node IP: `192.168.0.10`
- Kubernetes API endpoint: `https://192.168.0.10:6443`

## Project Structure

```
homelab-k8s/
├── .sops.yaml                    # SOPS configuration
├── age.agekey                    # Age private key (git-ignored)
└── infra/
    └── talos/
        ├── talconfig.yaml        # Talhelper configuration
        ├── talenv.sops.yaml      # Encrypted environment variables
        └── clusterconfig/        # Generated configs (git-ignored)
            ├── homelab-cluster-homelab-node-01.yaml
            └── talosconfig
```

## Initial Setup

### 1. Generate Age Key

```bash
# Generate a new age key pair
age-keygen -o age.agekey

# Extract the public key (for .sops.yaml)
age-keygen -y age.agekey
```

### 2. Configure SOPS

Create `.sops.yaml` in the repository root:

```yaml
creation_rules:
  - path_regex: infra/talos/talenv.sops.yaml
    encrypted_regex: ^(cluster|certs|secrets|trustdinfo)$
    key_groups:
      - age:
          - <your-age-public-key>
```

### 3. Create Environment Variables File

Create `infra/talos/talenv.sops.yaml` with your Tailscale auth key:

```yaml
TAILSCALE_AUTH_KEY: tskey-auth-xxxxx
```

Encrypt it with SOPS:

```bash
sops -e -i infra/talos/talenv.sops.yaml
```

### 4. Generate Talos Secrets

> ⚠️ **CRITICAL**: This step is mandatory. Without `talsecret.sops.yaml`, running
> `talhelper genconfig` will generate new certificates each time, which will
> **invalidate access to existing clusters**.

```bash
cd infra/talos

# Generate cluster secrets (first time only)
talhelper gensecret > talsecret.sops.yaml

# Encrypt the secrets
sops -e -i talsecret.sops.yaml

# Commit the encrypted secrets to Git
git add talsecret.sops.yaml
git commit -m "chore: add encrypted Talos secrets"
```

### 5. Generate Talos Configuration

> ⚠️ **WARNING**: Before running `talhelper genconfig`, ensure that
> `talsecret.sops.yaml` exists. If it doesn't exist, the command will generate
> new certificates and you will lose access to any existing cluster.

```bash
cd infra/talos

# Set the age key for SOPS decryption
export SOPS_AGE_KEY_FILE=/path/to/age.agekey

# Verify talsecret.sops.yaml exists
ls talsecret.sops.yaml || echo "ERROR: talsecret.sops.yaml not found!"

# Generate Talos configs (--no-gitignore to allow tracking talosconfig)
talhelper genconfig --no-gitignore
```

This generates:
- `clusterconfig/homelab-cluster-homelab-node-01.yaml` - Node configuration (git-ignored via root .gitignore)
- `clusterconfig/talosconfig` - Talosctl client configuration (tracked in Git)

After generation, commit the `talosconfig`:

```bash
git add clusterconfig/talosconfig
git commit -m "chore: add talosconfig for cluster access"
```

## Boot and Install Talos

### 1. Download Talos ISO

Download the ISO from Image Factory with required extensions:

```
https://factory.talos.dev/image/e2e3b54334c85fdef4d78e88f880d185e0ce0ba0c9b5861bb5daa1cd6574db9b/v1.12.0/metal-amd64.iso
```

Or use the [Image Factory UI](https://factory.talos.dev/?arch=amd64&bootloader=auto&cmdline-set=true&extensions=-&extensions=siderolabs%2Fiscsi-tools&extensions=siderolabs%2Ftailscale&platform=metal&target=metal&version=1.12.0) to customize.

### 2. Boot from ISO

1. Create a bootable USB drive with the ISO
2. Boot the target machine from the USB
3. Note the IP address shown on the console (or use `192.168.0.10` if pre-configured)

### 3. Apply Configuration

```bash
cd infra/talos

# Apply the configuration to the node
talosctl apply-config --insecure \
  --nodes 192.168.0.10 \
  --file clusterconfig/homelab-cluster-homelab-node-01.yaml
```

### 4. Configure talosctl

Set the `TALOSCONFIG` environment variable to use the generated config:

```bash
export TALOSCONFIG=$(pwd)/clusterconfig/talosconfig
```

> **Note**: Unlike the [official Talos getting started guide](https://docs.siderolabs.com/talos/v1.12/getting-started/getting-started),
> you don't need to manually run `talosctl config endpoints` because Talhelper
> automatically includes the endpoints in the generated `talosconfig` file.

Verify the configuration:

```bash
cat clusterconfig/talosconfig | grep -A2 endpoints
# Should show: endpoints: [192.168.0.10]
```

### 5. Bootstrap the Cluster

Wait for the node to finish installing and reboot, then bootstrap etcd:

```bash
# Wait for the node to be ready (this may take a few minutes)
talosctl health --wait-timeout 10m

# Bootstrap etcd (only needed once, on the first control plane node)
talosctl bootstrap
```

> **Important**: Run `talosctl bootstrap` only ONCE on a SINGLE control plane node.

### 6. Get Kubernetes Access

```bash
# Merge your new cluster into your local Kubernetes configuration
talosctl kubeconfig --nodes 192.168.0.10 --talosconfig=./clusterconfig/talosconfig
```

## Post-Bootstrap: Install Cilium CNI via Flux

Since we disabled the default CNI (`cniConfig.name: none`), we need to install Cilium.
The cluster node will remain in `NotReady` state until a CNI is installed.

> **Note**: Cilium is now managed via Flux GitOps. The configuration is defined in `k8s/infrastructure/cilium/` and automatically reconciled by Flux.

### Bootstrap Flux (if not already done)

If Flux is not yet bootstrapped, follow the [Flux Bootstrap Guide](./flux-bootstrap.md) first.

### Verify Cilium Installation via Flux

After Flux is bootstrapped and reconciling, Cilium will be automatically installed. Verify the installation:

```bash
# Verify Flux is healthy
kubectl -n flux-system get pods
kubectl -n flux-system get kustomizations

# Check Flux sources and releases
kubectl -n flux-system get helmrepositories
kubectl -n kube-system get helmreleases

# Verify Cilium pods are running
kubectl -n kube-system get pods -l app.kubernetes.io/part-of=cilium

# Verify node becomes Ready
kubectl get nodes

# Check all pods are running
kubectl get pods -A
```

Expected output after successful installation:
- Flux `HelmRepository` and `HelmRelease` show Ready status
- Node status changes from `NotReady` to `Ready`
- CoreDNS pods transition from `Pending` to `Running`
- Cilium agent and operator pods are `Running`

### (Optional) Install Cilium CLI for Troubleshooting

```bash
brew install cilium-cli

# Check Cilium status
cilium status

# Run connectivity test (optional)
cilium connectivity test
```

### Rollback Procedure

If you need to rollback a Cilium configuration change:

1. Revert the Git commit that changed Cilium configuration:
   ```bash
   git revert <commit-hash>
   git push
   ```

2. Let Flux reconcile (or trigger reconciliation):
   ```bash
   flux reconcile kustomization infrastructure
   ```

3. Verify Cilium and node readiness:
   ```bash
   kubectl -n kube-system get helmreleases cilium
   kubectl -n kube-system get pods -l app.kubernetes.io/part-of=cilium
   kubectl get nodes
   ```

## Verify Tailscale Connection

```bash
# Check Tailscale extension is loaded
talosctl get extensions

# View Tailscale logs
talosctl logs ext-tailscale
```

## Troubleshooting

> **Note**: The following commands assume `TALOSCONFIG` is set.
> If not, add `--talosconfig=./clusterconfig/talosconfig` to each command.

### Check Node Status

```bash
talosctl get members
talosctl health
```

### View Logs

```bash
# Kernel logs
talosctl dmesg

# Service logs
talosctl logs kubelet
talosctl logs etcd
```

### Reset Node (if needed)

```bash
# Reset the node to factory state
talosctl reset --graceful=false --reboot
```

## Configuration Reference

### talconfig.yaml

| Field | Value | Description |
|-------|-------|-------------|
| `clusterName` | `homelab-cluster` | Kubernetes cluster name |
| `talosVersion` | `v1.12.0` | Talos Linux version |
| `kubernetesVersion` | `v1.35.0` | Kubernetes version |
| `endpoint` | `https://192.168.0.10:6443` | Kubernetes API endpoint |
| `cniConfig.name` | `none` | Disable default CNI for Cilium |
| `cluster.proxy.disabled` | `true` | Disable kube-proxy (Cilium replaces it) |
| `allowSchedulingOnControlPlanes` | `true` | Allow workloads on control plane (single node) |

### Cilium Configuration

| Setting | Value | Description |
|---------|-------|-------------|
| `version` | `1.18.5` | Cilium version |
| `ipam.mode` | `kubernetes` | Use Kubernetes-native IPAM |
| `kubeProxyReplacement` | `true` | Replace kube-proxy with eBPF |
| `k8sServiceHost` | `localhost` | KubePrism endpoint |
| `k8sServicePort` | `7445` | KubePrism port |
| `operator.replicas` | `1` | Single replica for single-node cluster |

### Image Factory Schematic

- **Schematic ID**: `e2e3b54334c85fdef4d78e88f880d185e0ce0ba0c9b5861bb5daa1cd6574db9b`
- **Extensions**:
  - `siderolabs/iscsi-tools` (for Longhorn storage)
  - `siderolabs/tailscale`
- **Image Factory URL**: [View/Edit Schematic](https://factory.talos.dev/?arch=amd64&bootloader=auto&cmdline-set=true&extensions=-&extensions=siderolabs%2Fiscsi-tools&extensions=siderolabs%2Ftailscale&platform=metal&target=metal&version=1.12.0)

> **Note**: For Day 2 operations such as upgrades and adding extensions, see [Talos Operations Guide](./talos-operations.md).
