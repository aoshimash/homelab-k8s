# Implementation Plan: Renovate導入

**Branch**: `008-renovate` | **Date**: 2026-01-03 | **Spec**: [spec.md](./spec.md)
**Input**: Feature specification from `/specs/008-renovate/spec.md`

## Summary

Homelab Kubernetesリポジトリに依存関係自動更新ツールRenovateを導入し、HelmRelease、GitHub Actions、およびコアコンポーネント（Talos/Flux）のバージョン更新を自動検出してPRを作成する。GitHub Appとしてマネージド版Renovateを使用し、`renovate.json`で設定を管理する。

## Technical Context

**Language/Version**: JSON5 (renovate.json5)  
**Primary Dependencies**: GitHub Renovate App (マネージド版)  
**Storage**: N/A (設定ファイルのみ)  
**Testing**: Renovate Dry-run、手動検証  
**Target Platform**: GitHub.com  
**Project Type**: Configuration-only (新規コード不要)  
**Performance Goals**: 新バージョンリリースから24時間以内にPR作成  
**Constraints**: 同時オープンPR数最大10個、自動マージなし  
**Scale/Scope**: HelmRelease 6個、GitHub Actions workflows 1個、SOPS暗号化ファイル 7個を除外

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-check after Phase 1 design.*

| Principle | Status | Notes |
|-----------|--------|-------|
| I. GitOps-First | ✅ PASS | Renovateは設定ファイルをGitで管理、PRを通じて変更を提案 |
| II. Declarative Configuration | ✅ PASS | renovate.json5で宣言的に設定を管理 |
| III. Secrets Management | ✅ PASS | SOPSファイルは明示的に除外設定（ignorePaths） |
| IV. Immutable Infrastructure | ✅ PASS | インフラ変更は発生しない（設定ファイルのみ） |
| V. Documentation as Code | ✅ PASS | docs/renovate.mdで運用手順を文書化予定 |

**Gate Result**: ✅ ALL PASS - Phase 0に進行可能

## Project Structure

### Documentation (this feature)

```text
specs/008-renovate/
├── plan.md              # This file
├── research.md          # Renovate設定のベストプラクティス調査
├── data-model.md        # renovate.json5の構造定義
├── quickstart.md        # セットアップ手順
├── contracts/           # Renovate動作の契約定義
│   └── reconciliation.md
└── tasks.md             # 実装タスク（/speckit.tasks で生成）
```

### Source Code (repository root)

```text
# Configuration files (新規作成)
renovate.json5           # Renovate設定ファイル

# Documentation (新規作成)
docs/
└── renovate.md          # 運用ドキュメント

# 更新対象ファイル（参照のみ、Renovateが更新）
k8s/infrastructure/
├── cilium/helmrelease.yaml
├── longhorn/helmrelease.yaml
├── tailscale/helmrelease.yaml
├── grafana-alloy/helmrelease.yaml
├── kube-state-metrics/helmrelease.yaml
└── metrics-server/helmrelease.yaml

.github/workflows/
└── k8s-lint-security.yaml

# 除外対象（SOPS暗号化ファイル）
*.sops.yaml              # ignorePaths で除外
k8s/flux/flux-system/gotk-components.yaml  # Flux自動生成
```

**Structure Decision**: Configuration-onlyプロジェクトのため、`renovate.json5`をリポジトリルートに配置。ソースコードの追加は不要。

## Constitution Check (Post-Design)

*Re-evaluation after Phase 1 design completion*

| Principle | Status | Verification |
|-----------|--------|--------------|
| I. GitOps-First | ✅ PASS | `renovate.json5`はGitで管理、PRを通じた変更提案 |
| II. Declarative Configuration | ✅ PASS | JSON5形式で宣言的に設定、Kustomize/Helmとの整合性あり |
| III. Secrets Management | ✅ PASS | `ignorePaths`で7個のSOPSファイルを除外設定済み |
| IV. Immutable Infrastructure | ✅ PASS | クラスターノードへの変更なし |
| V. Documentation as Code | ✅ PASS | `quickstart.md`作成済み、`docs/renovate.md`を実装タスクに含める |

**Post-Design Gate Result**: ✅ ALL PASS - Phase 2（タスク生成）に進行可能

## Generated Artifacts

| Artifact | Path | Status |
|----------|------|--------|
| Research | `specs/008-renovate/research.md` | ✅ Complete |
| Data Model | `specs/008-renovate/data-model.md` | ✅ Complete |
| Contracts | `specs/008-renovate/contracts/reconciliation.md` | ✅ Complete |
| Quickstart | `specs/008-renovate/quickstart.md` | ✅ Complete |

## Complexity Tracking

> 該当なし - Constitution Check違反なし
