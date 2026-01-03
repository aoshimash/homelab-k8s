# Implementation Plan: K8s Lint & Security Checks

**Branch**: `007-k8s-lint-security` | **Date**: 2026-01-03 | **Spec**: [spec.md](./spec.md)
**Input**: Feature specification from `/specs/007-k8s-lint-security/spec.md`

## Summary

ローカルとCIで一貫したK8sマニフェストのリントチェック（kubeconform）とセキュリティチェック（Trivy）を実現する。Taskfileでローカルタスクを統合し、GitHub Actionsで同一チェックをPR時に自動実行。失敗時はRequired status checkとしてマージをブロック。

## Technical Context

**Language/Version**: YAML, Taskfile (go-task v3)  
**Primary Dependencies**: kubeconform, trivy, go-task  
**Storage**: N/A  
**Testing**: Manual verification + CI  
**Target Platform**: GitHub Actions (ubuntu-latest), macOS/Linux (local)
**Project Type**: GitOps infrastructure repository  
**Performance Goals**: リント30秒以内、セキュリティ60秒以内  
**Constraints**: ローカルとCIで100%同一結果  
**Scale/Scope**: k8s/ディレクトリ配下全ファイル（約100ファイル）

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-check after Phase 1 design.*

| Principle | Status | Notes |
|-----------|--------|-------|
| I. GitOps-First | ✅ PASS | Taskfile、GitHub Actions Workflowは全てGitで管理 |
| II. Declarative Configuration | ✅ PASS | Taskfile.yaml、workflow YAMLは宣言的 |
| III. Secrets Management | ✅ PASS | シークレット不要（パブリックツールのみ使用） |
| IV. Immutable Infrastructure | ✅ PASS | クラスターノードへの変更なし |
| V. Documentation as Code | ✅ PASS | docs/にセットアップ手順を記載予定 |

**Gate Result**: ✅ ALL PASS - Phase 0 進行可

### Post-Design Re-check (Phase 1 完了後)

| Principle | Status | Notes |
|-----------|--------|-------|
| I. GitOps-First | ✅ PASS | Taskfile.yaml、.github/workflows/*.yamlは全てGitで管理 |
| II. Declarative Configuration | ✅ PASS | YAML形式の宣言的設定のみ使用 |
| III. Secrets Management | ✅ PASS | シークレット不要（パブリックツールのみ） |
| IV. Immutable Infrastructure | ✅ PASS | クラスターノード変更なし |
| V. Documentation as Code | ✅ PASS | docs/k8s-lint-security.md作成予定 |

**Post-Design Gate Result**: ✅ ALL PASS

## Project Structure

### Documentation (this feature)

```text
specs/007-k8s-lint-security/
├── plan.md              # This file
├── research.md          # Phase 0 output
├── data-model.md        # Phase 1 output
├── quickstart.md        # Phase 1 output
├── contracts/           # Phase 1 output
│   └── reconciliation.md
└── tasks.md             # Phase 2 output (via /speckit.tasks)
```

### Source Code (repository root)

```text
# New files to be created
Taskfile.yaml              # Task orchestration for local checks
.github/
└── workflows/
    └── k8s-lint-security.yaml  # GitHub Actions workflow

# Documentation
docs/
└── k8s-lint-security.md   # Setup and usage guide
```

**Structure Decision**: GitOpsリポジトリのため、src/は不要。Taskfile.yamlをルートに配置し、.github/workflows/にCIワークフローを追加。

## Complexity Tracking

> No violations - Constitution check passed.
