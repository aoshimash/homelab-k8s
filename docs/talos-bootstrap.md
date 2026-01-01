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

Since we disabled the default CNI (`cniConfig.name: none`), we need to install Cilium.
The cluster node will remain in `NotReady` state until a CNI is installed.

> **Note**: We use Helm for installation to facilitate future GitOps management with Flux or ArgoCD.

References:
- [Talos Kubernetes Guides - Deploy Cilium CNI](https://docs.siderolabs.com/kubernetes-guides/cni/deploying-cilium)
- [Cilium Installation using Helm](https://docs.cilium.io/en/stable/installation/k8s-install-helm/)

### 1. Add Cilium Helm Repository

```bash
helm repo add cilium https://helm.cilium.io/
helm repo update
```

### 2. Install Cilium

Install Cilium with Talos-specific configuration and kube-proxy replacement:

```bash
helm install cilium cilium/cilium --version 1.18.5 \
  --namespace kube-system \
  --set ipam.mode=kubernetes \
  --set kubeProxyReplacement=true \
  --set securityContext.capabilities.ciliumAgent="{CHOWN,KILL,NET_ADMIN,NET_RAW,IPC_LOCK,SYS_ADMIN,SYS_RESOURCE,DAC_OVERRIDE,FOWNER,SETGID,SETUID}" \
  --set securityContext.capabilities.cleanCiliumState="{NET_ADMIN,SYS_ADMIN,SYS_RESOURCE}" \
  --set cgroup.autoMount.enabled=false \
  --set cgroup.hostRoot=/sys/fs/cgroup \
  --set k8sServiceHost=localhost \
  --set k8sServicePort=7445 \
  --set operator.replicas=1
```

> **Note**: `operator.replicas=1` is set for single-node clusters. The default value is 2, but with pod anti-affinity rules, the second replica cannot be scheduled on the same node.

**Configuration explained:**

| Setting | Description |
|---------|-------------|
| `ipam.mode=kubernetes` | Use Kubernetes-native IPAM for pod IP allocation |
| `kubeProxyReplacement=true` | Replace kube-proxy with Cilium's eBPF-based implementation |
| `securityContext.capabilities.*` | Talos-specific: Drop `SYS_MODULE` capability (Talos doesn't allow kernel module loading) |
| `cgroup.autoMount.enabled=false` | Talos already mounts cgroupv2 |
| `cgroup.hostRoot=/sys/fs/cgroup` | Talos cgroupv2 mount path |
| `k8sServiceHost=localhost` | Use KubePrism (Talos local Kubernetes API proxy) |
| `k8sServicePort=7445` | KubePrism default port |
| `operator.replicas=1` | Single replica for single-node cluster (default is 2) |

### 3. Verify Installation

Wait for Cilium to be ready:

```bash
# Check Cilium pods
kubectl -n kube-system get pods -l app.kubernetes.io/part-of=cilium

# Verify node becomes Ready
kubectl get nodes

# Check all pods are running
kubectl get pods -A
```

Expected output after successful installation:
- Node status changes from `NotReady` to `Ready`
- CoreDNS pods transition from `Pending` to `Running`
- Cilium agent and operator pods are `Running`

### 4. (Optional) Delete kube-proxy

Since we enabled `kubeProxyReplacement=true`, kube-proxy is no longer needed:

```bash
kubectl -n kube-system delete daemonset kube-proxy
```

### 5. (Optional) Install Cilium CLI for Troubleshooting

```bash
brew install cilium-cli

# Check Cilium status
cilium status

# Run connectivity test (optional)
cilium connectivity test
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

- **Schematic ID**: `4a0d65c669d46663f377e7161e50cfd570c401f26fd9e7bda34a0216b6f1922b`
- **Extensions**: `siderolabs/tailscale`
- **Image Factory URL**: [View/Edit Schematic](https://factory.talos.dev/?arch=amd64&bootloader=auto&cmdline-set=true&extensions=-&extensions=siderolabs%2Ftailscale&platform=metal&target=metal&version=1.12.0)
