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

```bash
cd infra/talos

# Generate cluster secrets (first time only)
talhelper gensecret > talsecret.sops.yaml

# Encrypt the secrets
sops -e -i talsecret.sops.yaml
```

### 5. Generate Talos Configuration

```bash
cd infra/talos

# Set the age key for SOPS decryption
export SOPS_AGE_KEY_FILE=/path/to/age.agekey

# Generate Talos configs
talhelper genconfig
```

This generates:
- `clusterconfig/homelab-cluster-homelab-node-01.yaml` - Node configuration
- `clusterconfig/talosconfig` - Talosctl client configuration

## Boot and Install Talos

### 1. Download Talos ISO

Download the ISO from Image Factory with Tailscale extension:

```
https://factory.talos.dev/image/4a0d65c669d46663f377e7161e50cfd570c401f26fd9e7bda34a0216b6f1922b/v1.12.0/metal-amd64.iso
```

Or use the [Image Factory UI](https://factory.talos.dev/?arch=amd64&bootloader=auto&cmdline-set=true&extensions=-&extensions=siderolabs%2Ftailscale&platform=metal&target=metal&version=1.12.0) to customize.

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

## Post-Bootstrap: Install Cilium CNI

Since we disabled the default CNI (`cniConfig.name: none`), we need to install Cilium:

```bash
# Install Cilium CLI
brew install cilium-cli

# Install Cilium
cilium install --set ipam.mode=kubernetes

# Verify Cilium is running
cilium status --wait
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
| `allowSchedulingOnControlPlanes` | `true` | Allow workloads on control plane (single node) |

### Image Factory Schematic

- **Schematic ID**: `4a0d65c669d46663f377e7161e50cfd570c401f26fd9e7bda34a0216b6f1922b`
- **Extensions**: `siderolabs/tailscale`
- **Image Factory URL**: [View/Edit Schematic](https://factory.talos.dev/?arch=amd64&bootloader=auto&cmdline-set=true&extensions=-&extensions=siderolabs%2Ftailscale&platform=metal&target=metal&version=1.12.0)
