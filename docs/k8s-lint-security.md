# K8s Lint & Security Checks

## Overview

This repository uses automated lint and security checks for Kubernetes manifests to ensure quality and security before deployment. Checks run both locally (via Taskfile) and in CI (via GitHub Actions) with identical configurations.

**Tools Used**:
- **kubeconform**: Kubernetes schema validation
- **Trivy**: Security vulnerability and misconfiguration scanning

## Local Setup

### Prerequisites

| Tool | Purpose | Install |
|------|---------|---------|
| [aqua](https://aquaproj.github.io/) | CLI version manager | See [aqua docs](https://aquaproj.github.io/docs/install) |
| go-task | Task orchestration | `brew install go-task` |
| kubeconform | K8s schema validation | `brew install kubeconform` |
| trivy | Security scanning | Managed by aqua (see `aqua.yaml`) |

### Quick Install (macOS)

```bash
brew install go-task kubeconform
aqua install  # installs trivy (version pinned in aqua.yaml)
```

### Verify Installation

```bash
task --version      # v3.x
kubeconform -v      # v0.6.x
trivy --version     # v0.69.x (managed by aqua)
```

## Local Usage

### Run All Checks

```bash
task check
```

This runs lint and security checks sequentially.

### Run Individual Checks

```bash
# Lint only (kubeconform)
task lint

# Security only (trivy HIGH/CRITICAL)
task security

# Security scan showing all severities
task security:all
```

## CI Integration

### Automatic Checks

CI checks run automatically when:
- A Pull Request is created or updated
- Changes are made to: `k8s/**`, `Taskfile.yaml`, or `.github/workflows/k8s-lint-security.yaml`

### Required Status Checks

The following checks must pass before merging a PR:
- ✅ `lint` - kubeconform schema validation
- ✅ `security` - Trivy security scan

### Viewing Results

1. Open the "Checks" tab in your Pull Request
2. Click on `lint` or `security` job
3. Review detailed error messages in the logs

## Local/CI Parity

To ensure identical results between local and CI environments:

### Configuration Alignment

| Setting | Local (Taskfile.yaml) | CI (GitHub Actions) | Status |
|---------|----------------------|---------------------|--------|
| SKIP_KINDS | `HelmRelease,HelmRepository,Kustomization,GitRepository,CiliumNetworkPolicy` | Same | ✅ Match |
| Severity Threshold | `HIGH,CRITICAL` | `HIGH,CRITICAL` | ✅ Match |
| Target Directory | `k8s/` | `k8s/` | ✅ Match |

### Version Alignment

Tool versions are pinned in `aqua.yaml` to ensure local/CI parity. Run `aqua install` to sync:

```bash
aqua install  # installs pinned versions from aqua.yaml
```

## Troubleshooting

### Issue: CRD Validation Errors

```text
k8s/flux/helmrelease.yaml - HelmRelease my-app: could not find schema
```

**Solution**: CRDs are excluded via `-skip-kinds`. If you add a new CRD, update the `SKIP_KINDS` variable in `Taskfile.yaml` and the workflow file.

### Issue: False Positive Security Findings

If Trivy flags legitimate configurations:

```bash
# Skip specific checks (use sparingly)
trivy config --skip-checks KSV001 k8s/
```

Always verify that the flagged configuration is truly necessary before skipping.

### Issue: Tool Version Mismatch

If local and CI results differ, sync tool versions:

```bash
# Sync trivy version from aqua.yaml
aqua install

# Check versions
kubeconform -v
trivy --version
```

## Excluded Resources

The following CRDs are excluded from kubeconform validation (no official Kubernetes schemas):

- `HelmRelease` (Flux CD)
- `HelmRepository` (Flux CD)
- `Kustomization` (Flux CD)
- `GitRepository` (Flux CD)
- `CiliumNetworkPolicy` (Cilium)

## Next Steps

1. Run `task check` before committing changes
2. Fix any errors reported
3. Create a PR and verify CI checks pass
4. Merge after all checks succeed
