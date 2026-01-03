# Research: K8s Lint & Security Checks

**Feature**: 007-k8s-lint-security
**Date**: 2026-01-03

## 1. kubeconform - Kubernetes Schema Validation

### Decision
kubeconform v0.6.x を使用してK8sマニフェストのスキーマ検証を行う。

### Rationale
- kubeval の後継ツールとして高速・活発に開発
- Kubernetes公式JSONスキーマに準拠
- CRD（Custom Resource Definition）のスキーマ検証もサポート
- GitHub Actions および ローカル環境で一貫して動作

### Alternatives Considered
1. **kubeval**: 開発停滞、新しいK8sバージョンのスキーマ対応が遅い
2. **kube-linter**: ベストプラクティスチェッカーだがスキーマ検証が弱い
3. **pluto**: 非推奨API検出専用で範囲が狭い

### Key Configuration

```bash
# 基本的な使用法
kubeconform -summary -output text k8s/**/*.yaml

# CRDスキップ（Flux、Cilium等のCRDは公式スキーマがない）
kubeconform -summary -skip-kinds HelmRelease,HelmRepository,Kustomization,GitRepository -output text k8s/**/*.yaml

# 厳格モード（未知のフィールドをエラー扱い）
kubeconform -strict -summary k8s/**/*.yaml
```

### CRD Handling Strategy
Flux CD、Cilium等のCRDは公式Kubernetesスキーマに含まれないため、`-skip-kinds`で除外する。

除外対象CRD:
- `HelmRelease` (Flux)
- `HelmRepository` (Flux)
- `Kustomization` (Flux)
- `GitRepository` (Flux)
- `CiliumNetworkPolicy` (Cilium)

## 2. Trivy - Security Scanning

### Decision
Trivy v0.50.x を使用してK8sマニフェストのセキュリティ脆弱性と設定ミスをスキャン。

### Rationale
- Aqua Security製のOSSで活発に開発
- K8sマニフェストの設定ミス（misconfiguration）検出に強い
- SARIF、JSON、Table形式など多様な出力フォーマット
- GitHub Actions との連携が容易

### Alternatives Considered
1. **kube-bench**: CIS Benchmarkチェックだがマニフェスト検証ではない
2. **checkov**: Pythonベースで依存が重い
3. **kubesec**: シンプルだが検出ルールが限定的

### Key Configuration

```bash
# K8s設定ファイルスキャン
trivy config --exit-code 1 --severity HIGH,CRITICAL k8s/

# 全ての深刻度を表示（情報として）
trivy config --exit-code 0 k8s/

# 特定チェックの無効化（必要に応じて）
trivy config --skip-checks KSV001,KSV003 k8s/
```

### Severity Handling

| Severity | Action | Rationale |
|----------|--------|-----------|
| CRITICAL | Block (exit 1) | 即座に修正が必要なセキュリティリスク |
| HIGH | Block (exit 1) | 重大なセキュリティリスク |
| MEDIUM | Warn (exit 0) | 認識して対応を検討 |
| LOW | Warn (exit 0) | 参考情報として表示 |

### Common Checks to Potentially Skip
homelabの特性上、一部のチェックは意図的に無効化する可能性がある：

- `KSV001`: Process can elevate its own privileges (場合によりLonghorn等で必要)
- `KSV003`: Default capabilities: some containers need specific caps

## 3. Taskfile (go-task) - Task Orchestration

### Decision
go-task v3 を使用してローカルチェックタスクを統合。

### Rationale
- YAMLベースでシンプルな記法
- クロスプラットフォーム対応（macOS、Linux）
- 依存関係の定義が可能
- 既存のMakefileより可読性が高い

### Alternatives Considered
1. **Makefile**: 広く普及しているが複雑なタスクは読みにくい
2. **Just**: Rust製で高速だがエコシステムが小さい
3. **Bash scripts**: 柔軟だが構造化が難しい

### Taskfile Structure

```yaml
version: '3'

vars:
  K8S_DIR: k8s
  SKIP_KINDS: HelmRelease,HelmRepository,Kustomization,GitRepository,CiliumNetworkPolicy

tasks:
  lint:
    desc: Run kubeconform to validate K8s manifests
    cmds:
      - kubeconform -summary -skip-kinds {{.SKIP_KINDS}} {{.K8S_DIR}}/**/*.yaml

  security:
    desc: Run Trivy security scan on K8s manifests
    cmds:
      - trivy config --exit-code 1 --severity HIGH,CRITICAL {{.K8S_DIR}}/

  check:
    desc: Run all checks (lint + security)
    deps: [lint, security]

  check:all:
    desc: Run all checks sequentially with detailed output
    cmds:
      - task: lint
      - task: security
```

## 4. GitHub Actions Workflow

### Decision
PR時にトリガーされるGitHub Actionsワークフローを作成。ローカルと同一のチェックを実行。

### Rationale
- PR時の自動チェックでマージ前の品質保証
- Required status checkによるマージブロック
- GitHub-hosted runnerで追加コスト不要

### Workflow Structure

```yaml
name: K8s Lint & Security

on:
  pull_request:
    paths:
      - 'k8s/**'
      - 'Taskfile.yaml'

jobs:
  lint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Install kubeconform
        run: |
          curl -L -o kubeconform.tar.gz https://github.com/yannh/kubeconform/releases/latest/download/kubeconform-linux-amd64.tar.gz
          tar xzf kubeconform.tar.gz
          sudo mv kubeconform /usr/local/bin/
      - name: Run kubeconform
        run: kubeconform -summary -skip-kinds HelmRelease,HelmRepository,Kustomization,GitRepository -output text k8s/**/*.yaml

  security:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Run Trivy
        uses: aquasecurity/trivy-action@master
        with:
          scan-type: 'config'
          scan-ref: 'k8s/'
          exit-code: '1'
          severity: 'HIGH,CRITICAL'
```

### Trigger Strategy

| Event | Paths | Action |
|-------|-------|--------|
| pull_request | k8s/**, Taskfile.yaml | チェック実行 |
| push (main) | k8s/** | チェック実行（オプション） |

### Local/CI Parity
ローカルとCIで同一結果を保証するため：
1. 同一ツールバージョンを使用（Taskfileでバージョン指定推奨）
2. 同一オプション・設定を使用
3. 同一対象ファイル（k8s/**/*.yaml）をスキャン

## 5. Tool Version Management

### Decision
ローカル環境でのツールインストールはbrewまたはaqua推奨。CIでは明示的にバージョン指定。

### Version Pinning Strategy

| Tool | Recommended Version | CI Installation |
|------|---------------------|-----------------|
| kubeconform | v0.6.x | GitHub release download |
| trivy | v0.50.x | aquasecurity/trivy-action |
| go-task | v3.x | brew install go-task |

### Local Installation (macOS)

```bash
# Homebrew
brew install kubeconform trivy go-task

# または aqua (version manager)
aqua install
```

## 6. Error Output Format

### Decision
人間が読みやすい形式（table/text）をデフォルトとし、CI用にJSONも対応。

### kubeconform Output

```text
Summary: 98 resources found parsing input - Valid: 95, Invalid: 2, Errors: 1, Skipped: 0
k8s/apps/myapp/deployment.yaml - Deployment nginx: is invalid: spec.template.spec.containers[0].image is required
```

### Trivy Output

```text
k8s/infrastructure/longhorn/helmrelease.yaml
============================================
Tests: 15 (SUCCESSES: 13, FAILURES: 2, EXCEPTIONS: 0)
Failures: 2 (HIGH: 1, CRITICAL: 1)

HIGH: Container should not be privileged
─────────────────────────────────────────
 Container 'longhorn-manager' of Deployment 'longhorn-manager' should set 'securityContext.privileged' to false
```

## Summary of Technical Decisions

| Topic | Decision |
|-------|----------|
| Lint Tool | kubeconform v0.6.x |
| Security Tool | Trivy v0.50.x (config scan) |
| Task Orchestration | go-task v3 (Taskfile.yaml) |
| CI Platform | GitHub Actions |
| CRD Handling | Skip FluxCD/Cilium CRDs |
| Severity Threshold | HIGH, CRITICAL → exit 1 |
| Trigger | PR時 + k8s/** パス変更時 |
| Version Management | brew/aqua (local), explicit versions (CI) |
