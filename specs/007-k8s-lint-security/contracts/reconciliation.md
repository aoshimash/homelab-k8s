# Contract: CI/CD Reconciliation

**Feature**: 007-k8s-lint-security
**Date**: 2026-01-03

## Overview

Pull Request時にK8sマニフェストのリントとセキュリティチェックを自動実行し、品質を保証する。

## Workflow Contract

### Trigger Events

| Event | Condition | Action |
|-------|-----------|--------|
| `pull_request` | paths: `k8s/**`, `Taskfile.yaml`, `.github/workflows/k8s-lint-security.yaml` | Run checks |

### Jobs

#### Job: lint

**Purpose**: Validate K8s manifests against Kubernetes JSON schemas

**Inputs**:
- `k8s/**/*.yaml` - All YAML files in k8s directory

**Outputs**:
- Exit code 0: All valid
- Exit code 1: Invalid resources found

**Contract**:

```yaml
# Guaranteed behavior
- MUST validate all .yaml files under k8s/
- MUST skip CRDs without official schemas (HelmRelease, etc.)
- MUST output summary with valid/invalid/skipped counts
- MUST fail (exit 1) if any standard K8s resource is invalid
```

#### Job: security

**Purpose**: Scan K8s manifests for security vulnerabilities and misconfigurations

**Inputs**:
- `k8s/` - K8s manifests directory

**Outputs**:
- Exit code 0: No HIGH/CRITICAL findings
- Exit code 1: HIGH or CRITICAL findings detected

**Contract**:

```yaml
# Guaranteed behavior
- MUST scan all manifests under k8s/
- MUST report HIGH and CRITICAL severity issues
- MUST fail (exit 1) on HIGH/CRITICAL findings
- MUST output table format with file, severity, and description
```

## Status Check Contract

### Required Status Checks

| Check Name | Required | Block Merge on Failure |
|------------|----------|------------------------|
| `lint` | Yes | Yes |
| `security` | Yes | Yes |

### Configuration (GitHub Repository Settings)

```
Branch protection rule for: main
├── Require status checks to pass before merging: ✓
│   ├── lint
│   └── security
└── Require branches to be up to date: ✓
```

## Local/CI Parity Contract

### Guarantee

ローカル環境とCI環境で同一のマニフェストに対して同一のチェック結果を得られることを保証する。

### Implementation

| Aspect | Local (task) | CI (GitHub Actions) | Parity |
|--------|--------------|---------------------|--------|
| kubeconform version | Latest (brew) | v0.6.7 (pinned) | バージョン差異の可能性あり |
| Trivy version | Latest (brew) | v0.28.0 action | バージョン差異の可能性あり |
| Skip kinds | Same list | Same list | ✅ |
| Severity threshold | HIGH,CRITICAL | HIGH,CRITICAL | ✅ |
| Target directory | k8s/ | k8s/ | ✅ |

### Version Alignment Strategy

ローカルでCIと完全同一の結果を得たい場合：

```bash
# Specific kubeconform version
curl -L -o kubeconform.tar.gz \
  "https://github.com/yannh/kubeconform/releases/download/v0.6.7/kubeconform-darwin-amd64.tar.gz"
tar xzf kubeconform.tar.gz
sudo mv kubeconform /usr/local/bin/

# Trivy - use aqua or specific version
aqua install trivy@0.50.0
```

## Error Handling Contract

### Lint Errors

```yaml
# Expected output format
Summary: X resources found parsing input - Valid: Y, Invalid: Z, Errors: E, Skipped: S

# Per-file errors
<path> - <Kind> <name>: is invalid: <field> <error message>
```

### Security Findings

```yaml
# Expected output format
<path>
============================================
Tests: X (SUCCESSES: Y, FAILURES: Z, EXCEPTIONS: E)
Failures: N (HIGH: H, CRITICAL: C)

<SEVERITY>: <Check title>
─────────────────────────────────────────
 <Description and remediation guidance>
```

## Acceptance Criteria

1. **AC-001**: PR作成時にlintジョブとsecurityジョブが自動実行される
2. **AC-002**: lintジョブ失敗時にPRがマージブロックされる
3. **AC-003**: securityジョブ失敗時にPRがマージブロックされる
4. **AC-004**: ローカルで`task check`を実行した結果とCI結果が一致する
5. **AC-005**: チェック結果に問題箇所と修正ガイダンスが含まれる
