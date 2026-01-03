# Tasks: K8s Lint & Security Checks

**Input**: Design documents from `/specs/007-k8s-lint-security/`
**Prerequisites**: plan.md, spec.md, research.md, data-model.md, contracts/reconciliation.md, quickstart.md

**Tests**: Not requested - manual verification via `task` commands and CI runs

**Organization**: Tasks are grouped by user story to enable independent implementation and testing of each story.

## Format: `[ID] [P?] [Story] Description`

- **[P]**: Can run in parallel (different files, no dependencies)
- **[Story]**: Which user story this task belongs to (e.g., US1, US2, US3)
- Include exact file paths in descriptions

## Path Conventions

This is a GitOps infrastructure repository:
- `Taskfile.yaml` - Root level task orchestration
- `.github/workflows/` - CI workflow definitions
- `docs/` - Documentation
- `k8s/` - Target K8s manifests (existing)

---

## Phase 1: Setup (Shared Infrastructure)

**Purpose**: Create basic Taskfile structure and GitHub workflows directory

- [x] T001 Create Taskfile.yaml skeleton with version and vars section in `Taskfile.yaml`
- [x] T002 [P] Create `.github/workflows/` directory structure

---

## Phase 2: Foundational (Blocking Prerequisites)

**Purpose**: No foundational tasks needed - this feature adds new tooling without dependencies on existing infrastructure

**⚠️ CRITICAL**: Skip to User Stories - no blocking prerequisites

**Checkpoint**: Foundation ready - user story implementation can now begin

---

## Phase 3: User Story 1 - ローカルでK8sマニフェストのリントチェックを実行する (Priority: P1) 🎯 MVP

**Goal**: 開発者がローカルで `task lint` を実行してkubeconformによるスキーマ検証ができる

**Independent Test**: `task lint` を実行し、k8s/配下のマニフェストがスキーマ検証される

### Implementation for User Story 1

- [ ] T003 [US1] Add SKIP_KINDS variable for CRDs (HelmRelease, etc.) in `Taskfile.yaml`
- [ ] T004 [US1] Implement `lint` task with kubeconform command in `Taskfile.yaml`
- [ ] T005 [US1] Test lint task locally with `task lint` command

**Checkpoint**: `task lint` が動作し、k8s/配下のマニフェストをスキーマ検証できる

---

## Phase 4: User Story 2 - ローカルでK8sマニフェストのセキュリティチェックを実行する (Priority: P1)

**Goal**: 開発者がローカルで `task security` を実行してTrivyによるセキュリティスキャンができる

**Independent Test**: `task security` を実行し、HIGH/CRITICAL脆弱性が検出される/されないことを確認

### Implementation for User Story 2

- [ ] T006 [US2] Implement `security` task with Trivy config scan in `Taskfile.yaml`
- [ ] T007 [US2] Implement `security:all` task showing all severities in `Taskfile.yaml`
- [ ] T008 [US2] Test security task locally with `task security` command

**Checkpoint**: `task security` が動作し、k8s/配下のマニフェストをセキュリティスキャンできる

---

## Phase 5: User Story 3 - ローカルで全チェックを一括実行する (Priority: P2)

**Goal**: 開発者が `task check` でリントとセキュリティチェックを一括実行できる

**Independent Test**: `task check` を実行し、lint → security の順で両方のチェックが実行される

### Implementation for User Story 3

- [ ] T009 [US3] Implement `check` task that runs lint and security sequentially in `Taskfile.yaml`
- [ ] T010 [US3] Test check task locally with `task check` command

**Checkpoint**: `task check` が動作し、全チェックを一括実行できる

---

## Phase 6: User Story 4 - CIでPull Request時にリントとセキュリティチェックを自動実行する (Priority: P1)

**Goal**: PR作成時にGitHub Actionsでリントとセキュリティチェックが自動実行される

**Independent Test**: テストPRを作成し、GitHub Actionsワークフローが実行されることを確認

### Implementation for User Story 4

- [ ] T011 [P] [US4] Create workflow file with trigger configuration in `.github/workflows/k8s-lint-security.yaml`
- [ ] T012 [US4] Add lint job with kubeconform installation and execution in `.github/workflows/k8s-lint-security.yaml`
- [ ] T013 [US4] Add security job with trivy-action in `.github/workflows/k8s-lint-security.yaml`
- [ ] T014 [US4] Test workflow by creating a test PR

**Checkpoint**: PRを作成するとGitHub Actionsでチェックが自動実行される

---

## Phase 7: User Story 5 - ローカルとCIで同一のチェックを実行する (Priority: P1)

**Goal**: ローカルとCIで同一のチェックロジック・オプションを使用し、結果の一致を保証

**Independent Test**: ローカルの `task lint` と CI の lint job が同一オプションを使用していることを確認

### Implementation for User Story 5

- [ ] T015 [US5] Verify SKIP_KINDS in Taskfile matches CI workflow skip-kinds in both files
- [ ] T016 [US5] Verify Trivy severity threshold (HIGH,CRITICAL) matches between local and CI in both files
- [ ] T017 [US5] Document version alignment strategy in `docs/k8s-lint-security.md`

**Checkpoint**: ローカルとCIのチェック設定が同一であることを確認済み

---

## Phase 8: Polish & Cross-Cutting Concerns

**Purpose**: Documentation and final validation

- [ ] T018 [P] Create documentation with prerequisites and usage guide in `docs/k8s-lint-security.md`
- [ ] T019 Update README.md with link to new documentation
- [ ] T020 Run full validation: `task check` locally and create test PR for CI
- [ ] T021 Configure branch protection rules for required status checks (manual via GitHub UI)

---

## Dependencies & Execution Order

### Phase Dependencies

- **Setup (Phase 1)**: No dependencies - can start immediately
- **Foundational (Phase 2)**: Skipped - no blocking prerequisites
- **User Stories (Phase 3-7)**: Can proceed in priority order
  - US1 and US2 can run in parallel (different tasks in same file)
  - US3 depends on US1 + US2 completion
  - US4 can run in parallel with US1-3 (different file)
  - US5 depends on US1-4 completion (verification task)
- **Polish (Phase 8)**: Depends on all user stories being complete

### User Story Dependencies

- **User Story 1 (P1)**: No dependencies - MVP starting point
- **User Story 2 (P1)**: No dependencies - can run parallel with US1
- **User Story 3 (P2)**: Depends on US1 + US2 (uses lint + security tasks)
- **User Story 4 (P1)**: No dependencies on local tasks - CI workflow is independent
- **User Story 5 (P1)**: Depends on US1, US2, US4 (verification that they match)

### Within Each User Story

- Configuration before testing
- Task implementation before verification
- Local validation before CI integration

### Parallel Opportunities

- T001 and T002 can run in parallel (Setup phase)
- T003-T005 (US1) and T011-T013 (US4) can run in parallel (different files)
- T006-T008 (US2) and T011-T013 (US4) can run in parallel (different files)
- T018-T019 (Polish) can run in parallel (different files)

---

## Parallel Example: Setup Phase

```bash
# Launch setup tasks together:
Task T001: "Create Taskfile.yaml skeleton"
Task T002: "Create .github/workflows/ directory"
```

## Parallel Example: User Story 1 + 4

```bash
# Launch local and CI tasks in parallel (different files):
Task T003-T005: "Implement lint task in Taskfile.yaml"
Task T011-T013: "Implement lint job in .github/workflows/k8s-lint-security.yaml"
```

---

## Implementation Strategy

### MVP First (User Story 1 Only)

1. Complete Phase 1: Setup (T001-T002)
2. Complete Phase 3: User Story 1 (T003-T005)
3. **STOP and VALIDATE**: Test `task lint` locally
4. Commit and verify lint functionality works

### Incremental Delivery

1. Setup → US1 (lint) → Test `task lint` ✓
2. Add US2 (security) → Test `task security` ✓
3. Add US3 (check) → Test `task check` ✓
4. Add US4 (CI) → Create test PR ✓
5. Add US5 (parity verification) → Confirm local = CI ✓
6. Polish → Documentation and branch protection

### Optimal Execution Order

For single developer:
1. T001 → T002 (Setup)
2. T003 → T004 → T005 (US1: lint)
3. T006 → T007 → T008 (US2: security)
4. T009 → T010 (US3: check)
5. T011 → T012 → T013 → T014 (US4: CI)
6. T015 → T016 → T017 (US5: parity)
7. T018 → T019 → T020 → T021 (Polish)

---

## Notes

- [P] tasks = different files, no dependencies
- [Story] label maps task to specific user story for traceability
- Each user story should be independently completable and testable
- Commit after each task or logical group
- Stop at any checkpoint to validate story independently
- Manual verification via `task` commands - no automated test suite
- Branch protection rules (T021) require GitHub UI configuration
