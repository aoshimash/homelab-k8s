# Tasks: Renovate導入

**Input**: Design documents from `/specs/008-renovate/`
**Prerequisites**: plan.md ✅, spec.md ✅, research.md ✅, data-model.md ✅, contracts/ ✅

**Tests**: Not requested - this is a configuration-only feature with manual verification

**Organization**: Tasks are grouped by user story to enable independent implementation and testing of each story.

## Format: `[ID] [P?] [Story] Description`

- **[P]**: Can run in parallel (different files, no dependencies)
- **[Story]**: Which user story this task belongs to (e.g., US1, US2, US3)
- Include exact file paths in descriptions

## Path Conventions

- Configuration files at repository root
- Documentation in `docs/`

---

## Phase 1: Setup (GitHub Renovate App)

**Purpose**: GitHub Renovate Appのインストールと基本設定

- [ ] T001 Install GitHub Renovate App from https://github.com/apps/renovate for aoshimash/homelab-k8s repository
- [ ] T002 Create renovate.json5 configuration file at repository root with base structure from research.md

---

## Phase 2: Foundational (Core Configuration)

**Purpose**: 全User Storyに共通する基盤設定

**⚠️ CRITICAL**: User Story固有の設定を追加する前に完了必須

- [ ] T003 Configure extends presets (config:recommended, helpers:pinGitHubActionDigests) in renovate.json5
- [ ] T004 Configure PR limits (prConcurrentLimit: 10, separateMajorMinor: true, separateMultipleMajor: true) in renovate.json5
- [ ] T005 Configure automerge: false in renovate.json5
- [ ] T006 [P] Configure ignorePaths for SOPS files (**/*.sops.yaml, .sops.yaml) in renovate.json5
- [ ] T007 [P] Configure ignorePaths for Flux generated file (k8s/flux/flux-system/gotk-components.yaml) in renovate.json5
- [ ] T008 [P] Configure labels (dependencies, renovate) in renovate.json5

**Checkpoint**: Base configuration complete - User Story specific settings can now be added

---

## Phase 3: User Story 1 - Helm Chart更新の自動検出 (Priority: P1) 🎯 MVP

**Goal**: HelmRelease（Cilium、Longhorn、Tailscale、Grafana Alloy、kube-state-metrics、metrics-server）のバージョン更新を自動検出しPRを作成

**Independent Test**: 
1. HelmReleaseのバージョンを意図的に古くする
2. Renovate Dashboardで検出を確認
3. 更新PRが自動作成されることを確認

### Implementation for User Story 1

- [ ] T009 [US1] Configure flux manager fileMatch for k8s/infrastructure/**/*.yaml in renovate.json5
- [ ] T010 [US1] Verify HelmRelease detection for cilium in k8s/infrastructure/cilium/helmrelease.yaml
- [ ] T011 [US1] Verify HelmRelease detection for longhorn in k8s/infrastructure/longhorn/helmrelease.yaml
- [ ] T012 [US1] Verify HelmRelease detection for tailscale in k8s/infrastructure/tailscale/helmrelease.yaml
- [ ] T013 [US1] Verify HelmRelease detection for grafana-alloy in k8s/infrastructure/grafana-alloy/helmrelease.yaml
- [ ] T014 [US1] Verify HelmRelease detection for kube-state-metrics in k8s/infrastructure/kube-state-metrics/helmrelease.yaml
- [ ] T015 [US1] Verify HelmRelease detection for metrics-server in k8s/infrastructure/metrics-server/helmrelease.yaml

**Checkpoint**: HelmRelease更新が検出され、PRが自動作成される状態

---

## Phase 4: User Story 2 - GitHub Actions更新の自動検出 (Priority: P2)

**Goal**: GitHub Actionsで使用するアクション（actions/checkout、trivy-action等）のバージョン更新を自動検出しPRを作成

**Independent Test**:
1. GitHub Actionsワークフローのアクションバージョンを確認
2. Renovate Dashboardで検出を確認
3. 更新PRが自動作成されることを確認

### Implementation for User Story 2

- [ ] T016 [US2] Verify github-actions manager is enabled (included in config:recommended preset) in renovate.json5
- [ ] T017 [US2] Verify GitHub Actions detection for .github/workflows/k8s-lint-security.yaml
- [ ] T018 [US2] Confirm SHA pinning is applied via helpers:pinGitHubActionDigests preset

**Checkpoint**: GitHub Actions更新が検出され、SHAピン留め付きのPRが自動作成される状態

---

## Phase 5: User Story 3 - Talos/Fluxコンポーネントの更新通知 (Priority: P3)

**Goal**: Talos LinuxやFluxCDの新バージョンリリースを把握し、計画的なアップグレードを可能にする

**Independent Test**:
1. Renovate Dashboardでコアコンポーネントの更新状況を確認
2. 手動アップグレードが必要な旨がPR/Issueに記載されることを確認

### Implementation for User Story 3

- [ ] T019 [US3] Add custom manager or packageRules for Talos version tracking in renovate.json5 (if detectable)
- [ ] T020 [US3] Verify Flux components are tracked via Dashboard (gotk-sync.yaml references)
- [ ] T021 [US3] Document manual upgrade process for Talos/Flux in docs/renovate.md

**Checkpoint**: Talos/Fluxの新バージョンが把握可能な状態

---

## Phase 6: Polish & Documentation

**Purpose**: ドキュメント整備と最終検証

- [ ] T022 [P] Create operational documentation in docs/renovate.md
- [ ] T023 [P] Update README.md with Renovate badge and brief description
- [ ] T024 Merge Onboarding PR created by Renovate
- [ ] T025 Validate all contracts in specs/008-renovate/contracts/reconciliation.md
- [ ] T026 Run quickstart.md validation checklist

---

## Dependencies & Execution Order

### Phase Dependencies

- **Setup (Phase 1)**: No dependencies - GitHub Appインストールから開始
- **Foundational (Phase 2)**: Setupの完了が必要 - 全User Storyをブロック
- **User Stories (Phase 3-5)**: Foundationalの完了が必要
  - 各User Storyは独立して検証可能
  - 優先順位順（P1 → P2 → P3）で実装推奨
- **Polish (Phase 6)**: 全User Story完了後

### User Story Dependencies

- **User Story 1 (P1)**: Phase 2完了後に開始可能 - 他Storyへの依存なし
- **User Story 2 (P2)**: Phase 2完了後に開始可能 - US1と並行可能
- **User Story 3 (P3)**: Phase 2完了後に開始可能 - US1/US2と並行可能

### Within Each User Story

- 設定追加 → 検出確認 → 検証完了の順
- 設定は累積的（renovate.json5に追記）

### Parallel Opportunities

- Phase 2: T006, T007, T008は並列実行可能（renovate.json5の異なるセクション）
- Phase 6: T022, T023は並列実行可能（異なるファイル）

---

## Parallel Example: Phase 2 Foundational

```bash
# 並列実行可能なタスク:
Task T006: "Configure ignorePaths for SOPS files"
Task T007: "Configure ignorePaths for Flux generated file"
Task T008: "Configure labels"
```

---

## Implementation Strategy

### MVP First (User Story 1 Only)

1. Complete Phase 1: GitHub Renovate Appインストール
2. Complete Phase 2: Core Configuration（全設定をrenovate.json5に追加）
3. Complete Phase 3: User Story 1（HelmRelease検出）
4. **STOP and VALIDATE**: 6つのHelmReleaseがDashboardに表示されることを確認
5. PRを作成してレビュー

### Incremental Delivery

1. Setup + Foundational → renovate.json5が動作可能
2. User Story 1追加 → HelmRelease更新検出（MVP!）
3. User Story 2追加 → GitHub Actions更新検出
4. User Story 3追加 → Talos/Flux更新通知
5. 各StoryはOnboarding PRマージ前に検証可能

### Single Developer Strategy

1. Phase 1-2を完了（約30分）
2. Phase 3を完了し検証（約15分）
3. Phase 4を完了し検証（約10分）
4. Phase 5を完了し検証（約15分）
5. Phase 6でドキュメント整備（約20分）
6. Onboarding PRをマージして本番稼働

---

## Notes

- [P] tasks = 異なるファイルまたはrenovate.json5の異なるセクション
- [Story] label = 対応するUser Storyを示す
- 各User Storyは独立して検証可能
- Renovate Dashboardで検出状況を確認
- Onboarding PRマージ前に全設定を検証
- 設定変更後はRenovateが自動的に再スキャン

---

## Task Summary

| Phase | Task Count | Description |
|-------|------------|-------------|
| Phase 1: Setup | 2 | GitHub App インストール |
| Phase 2: Foundational | 6 | Core Configuration |
| Phase 3: US1 (P1) | 7 | HelmRelease検出 |
| Phase 4: US2 (P2) | 3 | GitHub Actions検出 |
| Phase 5: US3 (P3) | 3 | Talos/Flux通知 |
| Phase 6: Polish | 5 | ドキュメント・検証 |
| **Total** | **26** | |

### Parallel Opportunities

- Phase 2: 3タスク並列可能（T006, T007, T008）
- Phase 6: 2タスク並列可能（T022, T023）

### MVP Scope

- **MVP**: Phase 1 + Phase 2 + Phase 3（User Story 1）
- **MVP Task Count**: 15タスク
- **MVP成果**: HelmReleaseの自動更新PR作成が動作
