# AGENTS.md

This file provides guidance to AI coding agents (Claude Code, Cursor, GitHub Copilot, etc.) when working with code in this repository.

## Project Overview

Homelab Kubernetes cluster managed as GitOps Infrastructure-as-Code. Single-node Talos Linux cluster with Flux CD for continuous reconciliation. All cluster state is defined in Git — manual `kubectl apply` or `helm install` for persistent changes is prohibited.

## Key Commands

```bash
# Run all lint + security checks (required before PRs)
task check

# Individual checks
task lint              # kubeconform schema validation
task security          # Trivy HIGH/CRITICAL scan
task security:all      # Trivy all severities (informational)

# Trigger podcast metadata update job
task radigo:update-metadata PROGRAM=audrey
```

**Prerequisites**: `brew install go-task kubeconform trivy`

## Architecture

### Flux CD Reconciliation Layers

Flux reconciles three layers from `k8s/` in dependency order:

1. **`k8s/infrastructure/`** — Cluster infrastructure (Helm-based): Cilium CNI, Longhorn storage, Tailscale operator, Grafana Alloy, kube-state-metrics, metrics-server, CloudNativePG, Actions Runner Controller
2. **`k8s/configs/`** — Post-infrastructure configuration: Tailscale ingress/proxy, PostgreSQL cluster + databases, ARC runner definitions. Depends on `infrastructure` and waits for Tailscale operator health.
3. **`k8s/apps/`** — User applications: Audiobookshelf (podcast recording/serving), Home Assistant, Vikunja. Depends on `configs`.

Orchestrated via `k8s/flux/` Kustomizations with explicit `dependsOn` chains.

### Infrastructure Components

Each infrastructure component in `k8s/infrastructure/<name>/` follows a standard pattern:
- `kustomization.yaml` — lists component resources
- `helmrepository.yaml` — Flux HelmRepository source
- `helmrelease.yaml` — Flux HelmRelease with pinned version and values
- `namespace.yaml` — dedicated namespace (when needed)
- `secret-*.sops.yaml` — SOPS-encrypted secrets (when needed)

### Node Configuration

Single node (`192.168.0.10`) running Talos Linux. Config managed via Talhelper:
- `infra/talos/talconfig.yaml` — node and cluster config
- `infra/talos/talsecret.sops.yaml` — encrypted cluster secrets
- Cilium as CNI (kube-proxy disabled), Tailscale extension baked into Talos image

## Secrets Management

All secrets use **SOPS + Age** encryption. Rules defined in `.sops.yaml`.

- Encrypted files must be named `*.sops.yaml`
- Pattern `k8s/.*\.sops\.yaml$` encrypts `data` and `stringData` fields
- Age key file: `age.agekey` (gitignored, never commit)
- Encrypt: `sops -e -i <file>` / Decrypt: `sops -d -i <file>`
- Flux Kustomizations reference `sops-age` secret for automatic decryption

## CI/CD

- **PR checks** (`.github/workflows/k8s-lint-security.yaml`): Runs `task check` on changes to `k8s/**`, `Taskfile.yaml`, or Trivy configs
- **Image builds**: Workflows for radigo and yt-dlp-rajiko container images
- **ARC runner tests**: Workflow to validate Actions Runner Controller setup
- **Renovate**: Automated dependency updates for Helm chart versions in `k8s/infrastructure/` and GitHub Action versions

## Conventions

- Helm charts are deployed exclusively through Flux HelmRelease resources, never imperative `helm install`
- Kustomize is used for all manifest composition — each directory with multiple resources has a `kustomization.yaml`
- CRDs skipped in kubeconform validation: `HelmRelease, HelmRepository, Kustomization, GitRepository, CiliumNetworkPolicy` (see `SKIP_KINDS` in Taskfile.yaml)
- Trivy skip rules: `KSV0014` (readOnlyRootFilesystem — audiobookshelf needs writable volumes), `KSV0009` (hostNetwork — Home Assistant needs it for Matter/mDNS)
- The `docs/` directory contains operational documentation for each component
- `specs/` is a historical archive of feature planning documents — read-only reference, do not edit
