# Data Model: Renovate設定

**Date**: 2026-01-03  
**Feature**: 008-renovate

## Overview

Renovateの設定は`renovate.json5`ファイルで定義する。このファイルはリポジトリルートに配置され、依存関係の検出・更新ルールを宣言的に記述する。

## Entity: Configuration (renovate.json5)

### Schema Structure

```
renovate.json5
├── $schema           # JSON Schema URL
├── extends[]         # ベースプリセット
├── prConcurrentLimit # 同時オープンPR上限
├── separateMajorMinor # メジャー/マイナー分離
├── separateMultipleMajor # 複数メジャー分離
├── automerge         # 自動マージ設定
├── ignorePaths[]     # 除外パスパターン
├── flux{}            # Flux manager設定
│   └── fileMatch[]   # 対象ファイルパターン
└── labels[]          # PRラベル
```

### Field Definitions

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `$schema` | string | No | JSON Schema URL（エディタ補完用） |
| `extends` | string[] | Yes | 継承するプリセット名 |
| `prConcurrentLimit` | integer | Yes | 同時にオープンできるPRの最大数（10） |
| `separateMajorMinor` | boolean | Yes | メジャーとマイナー/パッチを別PRにするか（true） |
| `separateMultipleMajor` | boolean | Yes | 複数のメジャー更新を別PRにするか（true） |
| `automerge` | boolean | Yes | 自動マージを有効にするか（false） |
| `ignorePaths` | string[] | Yes | 処理対象から除外するファイルパターン |
| `flux.fileMatch` | string[] | Yes | Flux managerが処理するファイルパターン |
| `labels` | string[] | No | 作成されるPRに付与するラベル |

### Validation Rules

1. `prConcurrentLimit`は1以上の整数であること
2. `extends`には有効なRenovateプリセット名のみ指定可能
3. `ignorePaths`はglob形式の文字列であること
4. `flux.fileMatch`は正規表現形式の文字列であること

## Entity: Dependency (検出対象)

Renovateが自動検出する依存関係のタイプ。

### HelmRelease Dependencies

| Field | Source Location | Example |
|-------|-----------------|---------|
| Chart Name | `spec.chart.spec.chart` | `cilium` |
| Chart Version | `spec.chart.spec.version` | `1.18.5` |
| Repository | `spec.chart.spec.sourceRef.name` | `cilium` |

**対象ファイル**:
- `k8s/infrastructure/cilium/helmrelease.yaml`
- `k8s/infrastructure/longhorn/helmrelease.yaml`
- `k8s/infrastructure/tailscale/helmrelease.yaml`
- `k8s/infrastructure/grafana-alloy/helmrelease.yaml`
- `k8s/infrastructure/kube-state-metrics/helmrelease.yaml`
- `k8s/infrastructure/metrics-server/helmrelease.yaml`

### GitHub Actions Dependencies

| Field | Source Location | Example |
|-------|-----------------|---------|
| Action Name | `uses:` directive | `actions/checkout` |
| Action Version | `@` suffix | `v4` |
| Commit SHA | Pinned digest | `abc123...` |

**対象ファイル**:
- `.github/workflows/k8s-lint-security.yaml`

## Entity: Update PR (出力)

Renovateが作成するPull Requestの構造。

### PR Metadata

| Field | Description | Example |
|-------|-------------|---------|
| Title | 更新内容のサマリー | `chore(deps): update helm release cilium to v1.18.6` |
| Branch | 更新用ブランチ名 | `renovate/cilium-1.x` |
| Labels | 付与されるラベル | `dependencies`, `renovate` |
| Body | 変更詳細とリリースノート | Markdown形式 |

### PR Body Content

```markdown
## [Chart/Action Name] vX.Y.Z → vA.B.C

### Release Notes
[リリースノートへのリンク]

### Configuration
- **Schedule**: At any time
- **Automerge**: Disabled

---
This PR was created by [Renovate](https://github.com/renovatebot/renovate).
```

## State Diagram: PR Lifecycle

```
[New Version Detected]
         │
         ▼
    ┌─────────┐
    │ PR Open │
    └────┬────┘
         │
    ┌────┴────┐
    │         │
    ▼         ▼
┌───────┐ ┌────────┐
│Merged │ │ Closed │
└───────┘ └────────┘
    │         │
    ▼         ▼
[Applied] [New PR on next version]
```

## Relationships

```
Configuration (renovate.json5)
    │
    ├── extends → Preset[]
    │
    ├── ignorePaths → ExcludedFile[]
    │
    ├── flux.fileMatch → HelmRelease[]
    │
    └── github-actions → Workflow[]

HelmRelease
    │
    └── generates → Update PR

Workflow
    │
    └── generates → Update PR
```

## File Exclusion Matrix

| Pattern | Matches | Reason |
|---------|---------|--------|
| `**/*.sops.yaml` | 6 files | SOPS暗号化ファイル |
| `.sops.yaml` | 1 file | ルートSOPS設定 |
| `k8s/flux/flux-system/gotk-components.yaml` | 1 file | Flux自動生成 |
