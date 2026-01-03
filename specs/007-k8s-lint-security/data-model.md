# Data Model: K8s Lint & Security Checks

**Feature**: 007-k8s-lint-security
**Date**: 2026-01-03

## Configuration Files

### 1. Taskfile.yaml (Repository Root)

```yaml
version: '3'

vars:
  K8S_DIR: k8s
  # CRDs that don't have official Kubernetes schemas
  SKIP_KINDS: HelmRelease,HelmRepository,Kustomization,GitRepository,CiliumNetworkPolicy

tasks:
  lint:
    desc: Validate K8s manifests using kubeconform
    cmds:
      - |
        kubeconform \
          -summary \
          -skip-kinds {{.SKIP_KINDS}} \
          -output text \
          {{.K8S_DIR}}/**/*.yaml

  security:
    desc: Run Trivy security scan on K8s manifests
    cmds:
      - |
        trivy config \
          --exit-code 1 \
          --severity HIGH,CRITICAL \
          {{.K8S_DIR}}/

  security:all:
    desc: Run Trivy security scan showing all severities (informational)
    cmds:
      - |
        trivy config \
          --exit-code 0 \
          {{.K8S_DIR}}/

  check:
    desc: Run all checks (lint + security)
    cmds:
      - task: lint
      - task: security
```

**Purpose**: Orchestrate local lint and security checks with consistent commands.

**Tasks**:

| Task | Description | Exit Code |
|------|-------------|-----------|
| `lint` | Run kubeconform schema validation | 0 on success, non-zero on error |
| `security` | Run Trivy security scan (HIGH/CRITICAL only) | 0 on success, 1 on findings |
| `security:all` | Run Trivy showing all severities | Always 0 (informational) |
| `check` | Run lint + security sequentially | Non-zero if any fails |

### 2. GitHub Actions Workflow

**Path**: `.github/workflows/k8s-lint-security.yaml`

```yaml
name: K8s Lint & Security

on:
  pull_request:
    paths:
      - 'k8s/**'
      - 'Taskfile.yaml'
      - '.github/workflows/k8s-lint-security.yaml'

jobs:
  lint:
    name: Lint K8s Manifests
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Install kubeconform
        run: |
          KUBECONFORM_VERSION="v0.6.7"
          curl -L -o kubeconform.tar.gz \
            "https://github.com/yannh/kubeconform/releases/download/${KUBECONFORM_VERSION}/kubeconform-linux-amd64.tar.gz"
          tar xzf kubeconform.tar.gz
          sudo mv kubeconform /usr/local/bin/
          kubeconform -v

      - name: Run kubeconform
        run: |
          kubeconform \
            -summary \
            -skip-kinds HelmRelease,HelmRepository,Kustomization,GitRepository,CiliumNetworkPolicy \
            -output text \
            k8s/**/*.yaml

  security:
    name: Security Scan
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Run Trivy config scan
        uses: aquasecurity/trivy-action@0.28.0
        with:
          scan-type: 'config'
          scan-ref: 'k8s/'
          exit-code: '1'
          severity: 'HIGH,CRITICAL'
          format: 'table'
```

**Purpose**: Automate lint and security checks on PR creation/update.

**Jobs**:

| Job | Description | Required |
|-----|-------------|----------|
| `lint` | Schema validation with kubeconform | Yes (status check) |
| `security` | Security scan with Trivy | Yes (status check) |

### 3. Documentation

**Path**: `docs/k8s-lint-security.md`

Documentation structure:

```markdown
# K8s Lint & Security Checks

## Overview
- Purpose of lint and security checks
- Tools used (kubeconform, Trivy)

## Local Setup

### Prerequisites
- go-task
- kubeconform
- trivy

### Installation (macOS)
```bash
brew install kubeconform trivy go-task
```

### Running Checks
```bash
task lint       # Run lint only
task security   # Run security only
task check      # Run all checks
```

## CI Integration
- Automatic on PR
- Required status check configuration

## Troubleshooting
- Common errors and solutions
```

## Data Flow

```
┌─────────────────────────────────────────────────────────────────────────┐
│                        Local Development                                 │
│                                                                         │
│  ┌──────────────┐     ┌──────────────────┐     ┌──────────────────┐   │
│  │   Developer  │────▶│   task check     │────▶│     Results      │   │
│  │              │     │                  │     │  (pass/fail)     │   │
│  └──────────────┘     └──────────────────┘     └──────────────────┘   │
│                              │                                          │
│                              ▼                                          │
│                    ┌─────────┴─────────┐                               │
│                    │                   │                               │
│              ┌─────┴─────┐       ┌─────┴─────┐                        │
│              │ task lint │       │task security│                       │
│              └─────┬─────┘       └─────┬─────┘                        │
│                    │                   │                               │
│                    ▼                   ▼                               │
│              ┌───────────┐       ┌───────────┐                        │
│              │kubeconform│       │   Trivy   │                        │
│              └───────────┘       └───────────┘                        │
└─────────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────┐
│                           GitHub CI                                      │
│                                                                         │
│  ┌──────────────┐     ┌──────────────────┐     ┌──────────────────┐   │
│  │  Pull Request│────▶│  GitHub Actions  │────▶│   Status Check   │   │
│  │   Created    │     │    Workflow      │     │  (pass/fail)     │   │
│  └──────────────┘     └──────────────────┘     └──────────────────┘   │
│                              │                                          │
│                              ▼                                          │
│                    ┌─────────┴─────────┐                               │
│                    │                   │                               │
│              ┌─────┴─────┐       ┌─────┴─────┐                        │
│              │ lint job  │       │security job│                       │
│              └─────┬─────┘       └─────┬─────┘                        │
│                    │                   │                               │
│                    ▼                   ▼                               │
│              ┌───────────┐       ┌───────────┐                        │
│              │kubeconform│       │trivy-action│                       │
│              └───────────┘       └───────────┘                        │
└─────────────────────────────────────────────────────────────────────────┘
```

## Tool Versions

| Tool | Version | Source |
|------|---------|--------|
| kubeconform | v0.6.7 | GitHub Releases |
| Trivy | v0.50.x | aquasecurity/trivy-action@0.28.0 |
| go-task | v3.x | Homebrew |

## File Targets

### Scan Scope

```text
k8s/
├── apps/                    # ✅ Scanned
│   └── **/*.yaml
├── configs/                 # ✅ Scanned
│   └── **/*.yaml
├── flux/                    # ✅ Scanned (CRDs skipped by kubeconform)
│   └── **/*.yaml
└── infrastructure/          # ✅ Scanned
    └── **/*.yaml
```

### Excluded Resources (kubeconform only)

| Kind | Reason |
|------|--------|
| HelmRelease | Flux CD CRD - no official schema |
| HelmRepository | Flux CD CRD - no official schema |
| Kustomization | Flux CD CRD - no official schema |
| GitRepository | Flux CD CRD - no official schema |
| CiliumNetworkPolicy | Cilium CRD - no official schema |

## Exit Codes

### kubeconform

| Exit Code | Meaning |
|-----------|---------|
| 0 | All resources valid |
| 1 | Invalid resources found |
| 2 | Parse error (invalid YAML) |

### Trivy

| Exit Code | Meaning |
|-----------|---------|
| 0 | No findings (or below threshold) |
| 1 | Findings at specified severity |

### Combined (task check)

| Exit Code | Meaning |
|-----------|---------|
| 0 | All checks passed |
| Non-zero | At least one check failed |
