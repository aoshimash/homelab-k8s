# Tasks: Apps Directory Refactor

**Input**: Design documents from `/specs/010-apps-refactor/`
**Prerequisites**: plan.md (required), spec.md (required for user stories), research.md, data-model.md, contracts/

**Tests**: Not included - manual verification via `kustomize build` and `flux reconcile`

**Organization**: Tasks are grouped by user story to enable independent implementation and testing of each story.

## Format: `[ID] [P?] [Story] Description`

- **[P]**: Can run in parallel (different files, no dependencies)
- **[Story]**: Which user story this task belongs to (e.g., US1, US2, US3)
- Include exact file paths in descriptions

## Path Conventions

- **Primary path**: `k8s/apps/` for Kubernetes manifests
- **CI/CD**: `.github/workflows/` for GitHub Actions

---

## Phase 1: Setup (Directory Structure Creation)

**Purpose**: Create new directory structure before file moves

- [ ] T001 Create app/ subdirectory at `k8s/apps/audiobookshelf/app/`
- [ ] T002 [P] Create shared-secrets/ subdirectory at `k8s/apps/audiobookshelf/shared-secrets/`
- [ ] T003 [P] Create radigo-recorder/ subdirectory structure at `k8s/apps/audiobookshelf/radigo-recorder/`
- [ ] T004 [P] Create metadata-updater/ subdirectory structure at `k8s/apps/audiobookshelf/metadata-updater/`

---

## Phase 2: Foundational (File Moves)

**Purpose**: Move all files to new locations using git mv to preserve history

**⚠️ CRITICAL**: All moves must complete before kustomization.yaml updates

### Core App Files

- [ ] T005 [P] Move deployment.yaml using `git mv k8s/apps/audiobookshelf/deployment.yaml k8s/apps/audiobookshelf/app/`
- [ ] T006 [P] Move service.yaml using `git mv k8s/apps/audiobookshelf/service.yaml k8s/apps/audiobookshelf/app/`
- [ ] T007 [P] Move ingress.yaml using `git mv k8s/apps/audiobookshelf/ingress.yaml k8s/apps/audiobookshelf/app/`
- [ ] T008 [P] Move pvc.yaml using `git mv k8s/apps/audiobookshelf/pvc.yaml k8s/apps/audiobookshelf/app/`

### Shared Secrets

- [ ] T009 [P] Move shared-secrets files using `git mv k8s/apps/radigo/shared-secrets/* k8s/apps/audiobookshelf/shared-secrets/`

### Radigo Recorder (from recording/)

- [ ] T010 [P] Move radigo-recorder base files using `git mv k8s/apps/radigo/recording/base/* k8s/apps/audiobookshelf/radigo-recorder/base/`
- [ ] T011 [P] Move radigo-recorder configmap using `git mv k8s/apps/radigo/recording/configmap-record-script.yaml k8s/apps/audiobookshelf/radigo-recorder/`
- [ ] T012 [P] Move radigo-recorder kustomization using `git mv k8s/apps/radigo/recording/kustomization.yaml k8s/apps/audiobookshelf/radigo-recorder/`
- [ ] T013 [P] Move radigo-recorder programs/audrey using `git mv k8s/apps/radigo/recording/programs/audrey/* k8s/apps/audiobookshelf/radigo-recorder/programs/audrey/`
- [ ] T014 [P] Move radigo-recorder programs/ijuin using `git mv k8s/apps/radigo/recording/programs/ijuin/* k8s/apps/audiobookshelf/radigo-recorder/programs/ijuin/`

### Metadata Updater (from metadata/)

- [ ] T015 [P] Move metadata-updater base files using `git mv k8s/apps/radigo/metadata/base/* k8s/apps/audiobookshelf/metadata-updater/base/`
- [ ] T016 [P] Move metadata-updater configmap using `git mv k8s/apps/radigo/metadata/configmap-metadata-script.yaml k8s/apps/audiobookshelf/metadata-updater/`
- [ ] T017 [P] Move metadata-updater kustomization using `git mv k8s/apps/radigo/metadata/kustomization.yaml k8s/apps/audiobookshelf/metadata-updater/`
- [ ] T018 [P] Move metadata-updater programs/audrey using `git mv k8s/apps/radigo/metadata/programs/audrey/* k8s/apps/audiobookshelf/metadata-updater/programs/audrey/`
- [ ] T019 [P] Move metadata-updater programs/ijuin using `git mv k8s/apps/radigo/metadata/programs/ijuin/* k8s/apps/audiobookshelf/metadata-updater/programs/ijuin/`

### Cleanup

- [ ] T020 Remove old radigo directory `k8s/apps/radigo/`

**Checkpoint**: All files moved to new locations

---

## Phase 3: User Story 1 - Unified Directory Structure (Priority: P1) 🎯 MVP

**Goal**: All resources deployed to audiobookshelf namespace organized under single directory hierarchy

**Independent Test**: `kustomize build k8s/apps/` succeeds and outputs all expected resources

### Implementation for User Story 1

- [ ] T021 [US1] Create new kustomization.yaml at `k8s/apps/audiobookshelf/app/kustomization.yaml` with resources: deployment.yaml, service.yaml, ingress.yaml, pvc.yaml
- [ ] T022 [US1] Update kustomization.yaml at `k8s/apps/audiobookshelf/kustomization.yaml` to reference: namespace.yaml, app/, shared-secrets/, radigo-recorder/, metadata-updater/
- [ ] T023 [US1] Update kustomization.yaml at `k8s/apps/kustomization.yaml` to reference only audiobookshelf/ (remove radigo/)
- [ ] T024 [US1] Verify kustomize build succeeds: `kustomize build k8s/apps/`

**Checkpoint**: Directory structure unified, kustomize build succeeds

---

## Phase 4: User Story 2 - Preserved Functionality (Priority: P1)

**Goal**: All existing functionality works exactly as before after refactoring

**Independent Test**: Flux reconciliation completes without errors, all resources deployed correctly

### Implementation for User Story 2

- [ ] T025 [US2] Verify shared-secrets kustomization.yaml paths are correct at `k8s/apps/audiobookshelf/shared-secrets/kustomization.yaml`
- [ ] T026 [US2] Verify radigo-recorder kustomization.yaml paths are correct at `k8s/apps/audiobookshelf/radigo-recorder/kustomization.yaml`
- [ ] T027 [US2] Verify metadata-updater kustomization.yaml paths are correct at `k8s/apps/audiobookshelf/metadata-updater/kustomization.yaml`
- [ ] T028 [US2] Verify all resource outputs match before/after: `kustomize build k8s/apps/ | kubectl diff -f -`

**Checkpoint**: All resources identical to before refactoring

---

## Phase 5: User Story 3 - Clear Separation of Concerns (Priority: P2)

**Goal**: Subdirectories clearly separate different components

**Independent Test**: Navigate to each subdirectory and verify only relevant resources exist

### Implementation for User Story 3

- [ ] T029 [US3] Verify app/ contains only: deployment.yaml, service.yaml, ingress.yaml, pvc.yaml, kustomization.yaml
- [ ] T030 [US3] Verify shared-secrets/ contains only: secret-ghcr.sops.yaml, secret-audiobookshelf-api.sops.yaml, kustomization.yaml
- [ ] T031 [US3] Verify radigo-recorder/ contains only: base/, programs/, configmap-record-script.yaml, kustomization.yaml
- [ ] T032 [US3] Verify metadata-updater/ contains only: base/, programs/, configmap-metadata-script.yaml, kustomization.yaml

**Checkpoint**: All subdirectories contain only their relevant resources

---

## Phase 6: Polish & Cross-Cutting Concerns

**Purpose**: CI/CD updates and final verification

- [ ] T033 Update skip-dirs in `.github/workflows/k8s-lint-security.yaml` from `apps/radigo/recording/programs,apps/radigo/metadata/programs` to `apps/audiobookshelf/radigo-recorder/programs,apps/audiobookshelf/metadata-updater/programs`
- [ ] T034 [P] Run kubeconform lint: verify CI passes
- [ ] T035 [P] Run trivy security scan: verify CI passes
- [ ] T036 Create PR and verify CI checks pass
- [ ] T037 After merge, verify Flux reconciliation: `flux reconcile kustomization apps --with-source`
- [ ] T038 Verify all resources in audiobookshelf namespace: `kubectl get all -n audiobookshelf`

---

## Dependencies & Execution Order

### Phase Dependencies

- **Setup (Phase 1)**: No dependencies - can start immediately
- **Foundational (Phase 2)**: Depends on Setup completion - BLOCKS all user stories
- **User Stories (Phase 3-5)**: All depend on Foundational phase completion
- **Polish (Phase 6)**: Depends on all user stories being complete

### User Story Dependencies

- **User Story 1 (P1)**: Can start after Foundational (Phase 2) - Creates new kustomization structure
- **User Story 2 (P1)**: Can start after US1 - Verifies functionality preservation
- **User Story 3 (P2)**: Can start after US1 - Verifies component separation

### Within Each Phase

- Phase 1: All tasks [P] can run in parallel (creating directories)
- Phase 2: All move tasks [P] can run in parallel (different source/destination)
- Phase 3: Sequential (T021 → T022 → T023 → T024)
- Phase 4-5: Can run in parallel after Phase 3
- Phase 6: T033 first, then [P] tasks, then T036 → T037 → T038

### Parallel Opportunities

```bash
# Phase 1: All directory creation in parallel
mkdir -p k8s/apps/audiobookshelf/{app,shared-secrets}
mkdir -p k8s/apps/audiobookshelf/radigo-recorder/{base,programs/audrey,programs/ijuin}
mkdir -p k8s/apps/audiobookshelf/metadata-updater/{base,programs/audrey,programs/ijuin}

# Phase 2: All file moves in parallel
git mv k8s/apps/audiobookshelf/deployment.yaml k8s/apps/audiobookshelf/app/
git mv k8s/apps/audiobookshelf/service.yaml k8s/apps/audiobookshelf/app/
git mv k8s/apps/audiobookshelf/ingress.yaml k8s/apps/audiobookshelf/app/
git mv k8s/apps/audiobookshelf/pvc.yaml k8s/apps/audiobookshelf/app/
# ... etc
```

---

## Implementation Strategy

### MVP First (User Stories 1 & 2)

1. Complete Phase 1: Setup (create directories)
2. Complete Phase 2: Foundational (move files)
3. Complete Phase 3: User Story 1 (update kustomization files)
4. Complete Phase 4: User Story 2 (verify functionality)
5. **STOP and VALIDATE**: `kustomize build k8s/apps/` succeeds
6. Create PR for review

### Full Delivery

1. Complete MVP (above)
2. Add Phase 5: User Story 3 (verify separation)
3. Add Phase 6: Polish (CI/CD updates)
4. Merge PR and verify Flux reconciliation

---

## Notes

- All file moves use `git mv` to preserve Git history
- SOPS encrypted secrets need no changes (path patterns still match)
- Kustomize relative paths must be updated after file moves
- CI skip-dirs must be updated to new paths
- Verify before merge: `kustomize build k8s/apps/` outputs identical resources
