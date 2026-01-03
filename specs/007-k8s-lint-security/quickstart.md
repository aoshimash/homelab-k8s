# Quickstart: K8s Lint & Security Checks

**Feature**: 007-k8s-lint-security
**Date**: 2026-01-03

## Prerequisites

### Required Tools

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
trivy --version     # 0.50.x
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

### Expected Output

#### Successful Run

```text
$ task check
task: [lint] kubeconform -summary -skip-kinds HelmRelease,HelmRepository,Kustomization,GitRepository,CiliumNetworkPolicy -output text k8s/**/*.yaml
Summary: 98 resources found parsing input - Valid: 98, Invalid: 0, Errors: 0, Skipped: 0

task: [security] trivy config --exit-code 1 --severity HIGH,CRITICAL k8s/

2026-01-03T12:00:00.000+0900    INFO    Misconfiguration scanning is enabled
2026-01-03T12:00:00.000+0900    INFO    Need to update the built-in policies
2026-01-03T12:00:00.000+0900    INFO    Detected config files    num=98

No misconfigurations with HIGH or CRITICAL severity found.
```

#### Failed Run (Lint)

```text
$ task lint
Summary: 98 resources found parsing input - Valid: 96, Invalid: 2, Errors: 0, Skipped: 0

k8s/apps/broken/deployment.yaml - Deployment broken-app is invalid: spec.template.spec.containers is required
k8s/apps/broken/service.yaml - Service broken-svc is invalid: spec.ports[0].port is required
task: Failed to run task "lint": exit status 1
```

#### Failed Run (Security)

```text
$ task security
k8s/infrastructure/dangerous/deployment.yaml
============================================
Tests: 15 (SUCCESSES: 13, FAILURES: 2, EXCEPTIONS: 0)
Failures: 2 (HIGH: 1, CRITICAL: 1)

CRITICAL: Container runs as root
─────────────────────────────────
 Container 'app' in Deployment 'dangerous-app' should set 'securityContext.runAsNonRoot' to true

HIGH: Container has no security context
─────────────────────────────────
 Container 'app' in Deployment 'dangerous-app' should set 'securityContext'

task: Failed to run task "security": exit status 1
```

## CI Integration

### Automatic Checks

CIチェックは以下の条件で自動実行されます：

- Pull Requestが作成または更新されたとき
- 対象パス: `k8s/**`, `Taskfile.yaml`, `.github/workflows/k8s-lint-security.yaml`

### Required Status Checks

PRをマージするには、以下のチェックが成功する必要があります：

- ✅ `lint` - kubeconform schema validation
- ✅ `security` - Trivy security scan

### Viewing Results

1. Pull Requestの「Checks」タブを開く
2. `lint` または `security` ジョブをクリック
3. ログで詳細なエラーメッセージを確認

## Common Issues & Solutions

### Issue: CRD Validation Errors

```text
k8s/flux/helmrelease.yaml - HelmRelease my-app: could not find schema
```

**Solution**: CRDは`-skip-kinds`で除外済みのはず。新しいCRDを追加した場合は`Taskfile.yaml`の`SKIP_KINDS`変数を更新。

### Issue: False Positive Security Findings

Trivyが正当な設定を誤検出する場合：

```bash
# 特定チェックをスキップ
trivy config --skip-checks KSV001 k8s/
```

ただし、スキップする前に本当に必要な設定か確認してください。

### Issue: Tool Version Mismatch

ローカルとCIで結果が異なる場合、ツールバージョンを確認：

```bash
# バージョン確認
kubeconform -v
trivy --version

# CIと同じバージョンをインストール
# kubeconform: v0.6.7
# trivy: v0.50.x
```

## Next Steps

1. 初めてマニフェストを変更する前に`task check`を実行
2. エラーがあれば修正
3. PRを作成してCIチェックを確認
4. 全チェック成功後にマージ
