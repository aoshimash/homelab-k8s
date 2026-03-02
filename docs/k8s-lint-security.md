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
| go-task | Task orchestration | `brew install go-task` |
| kubeconform | K8s schema validation | `brew install kubeconform` |
| trivy | Security scanning | `brew install trivy` |

### Quick Install (macOS)

```bash
brew install go-task kubeconform trivy
```

### Verify Installation

```bash
task --version      # v3.x
kubeconform -v      # v0.6.x
trivy --version     # 0.69.x
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

For exact parity, use the same tool versions locally as in CI:

```bash
# kubeconform: v0.6.7 (CI version)
curl -L -o kubeconform.tar.gz \
  "https://github.com/yannh/kubeconform/releases/download/v0.6.7/kubeconform-darwin-amd64.tar.gz"
tar xzf kubeconform.tar.gz
sudo mv kubeconform /usr/local/bin/

# Trivy: Use latest (CI uses v0.69.2)
brew install trivy
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

If local and CI results differ, check tool versions:

```bash
# Check versions
kubeconform -v
trivy --version

# Install CI versions
# kubeconform: v0.6.7
# trivy: v0.69.x
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
