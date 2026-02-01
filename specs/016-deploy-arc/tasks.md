# Tasks: Deploy Actions Runner Controller (ARC)

**Input**: Design documents from `/specs/016-deploy-arc/`
**Prerequisites**: plan.md (required), spec.md (required for user stories), research.md, data-model.md, contracts/

**Tests**: Manual verification via kubectl and GitHub Actions workflow trigger (no automated tests required)

**Organization**: Tasks are grouped by user story to enable independent implementation and testing of each story.

## Format: `[ID] [P?] [Story] Description`

- **[P]**: Can run in parallel (different files, no dependencies)
- **[Story]**: Which user story this task belongs to (e.g., US1, US2, US3)
- Include exact file paths in descriptions

## Path Conventions

- **Infrastructure manifests**: `k8s/infrastructure/actions-runner-controller/`
- **Config manifests**: `k8s/configs/arc-runners/`
- **Flux kustomizations**: `k8s/flux/`

---

## Phase 1: Setup (Shared Infrastructure)

**Purpose**: Create directory structure and shared resources for ARC deployment

- [x] T001 Create controller directory structure at k8s/infrastructure/actions-runner-controller/
- [x] T002 Create runner config directory structure at k8s/configs/arc-runners/
- [x] T003 [P] Create HelmRepository for ARC OCI charts in k8s/infrastructure/actions-runner-controller/helmrepository.yaml

---

## Phase 2: Foundational (Blocking Prerequisites)

**Purpose**: GitHub App setup and namespace creation - MUST be complete before ANY user story

**⚠️ CRITICAL**: No user story work can begin until GitHub App credentials are available

- [x] T004 Create GitHub App in organization with required permissions (Self-hosted runners: Read/Write, Metadata: Read-only)
- [x] T005 Install GitHub App to target organization and note App ID and Installation ID
- [x] T006 Generate and download GitHub App private key (.pem file)
- [x] T007 [P] Create arc-systems namespace manifest in k8s/infrastructure/actions-runner-controller/namespace.yaml
- [x] T008 [P] Create arc-runners namespace manifest in k8s/configs/arc-runners/namespace.yaml
- [x] T009 Create SOPS-encrypted secret with GitHub App credentials in k8s/configs/arc-runners/secret-github-app.sops.yaml

**Checkpoint**: Foundation ready - GitHub App configured, namespaces defined, secret encrypted

---

## Phase 3: User Story 1 - Run GitHub Actions Workflows on Self-Hosted Runners (Priority: P1) 🎯 MVP

**Goal**: Deploy ARC controller and runner scale set so workflows can execute on homelab cluster

**Independent Test**: Trigger a workflow with `runs-on: homelab` and verify it completes successfully

### Implementation for User Story 1

- [x] T010 [US1] Create HelmRelease for ARC controller in k8s/infrastructure/actions-runner-controller/helmrelease.yaml
- [x] T011 [US1] Create Kustomization for controller directory in k8s/infrastructure/actions-runner-controller/kustomization.yaml
- [x] T012 [US1] Update infrastructure kustomization to include ARC controller in k8s/infrastructure/kustomization.yaml
- [x] T013 [US1] Create HelmRelease for runner scale set with homelab label in k8s/configs/arc-runners/helmrelease.yaml
- [x] T014 [US1] Create Kustomization for arc-runners directory in k8s/configs/arc-runners/kustomization.yaml
- [x] T015 [US1] Update configs kustomization to include arc-runners in k8s/configs/kustomization.yaml
- [x] T016 [US1] Commit and push changes to trigger FluxCD reconciliation
- [ ] T017 [US1] Verify controller pod is running in arc-systems namespace
- [ ] T018 [US1] Verify listener pod is running in arc-runners namespace
- [ ] T019 [US1] Create test workflow in a repository with runs-on: homelab
- [ ] T020 [US1] Trigger test workflow and verify runner pod is created and job completes

**Checkpoint**: User Story 1 complete - workflows can run on self-hosted runners

---

## Phase 4: User Story 2 - Automatic Runner Scaling (Priority: P2)

**Goal**: Verify scaling behavior works correctly (0-3 runners based on demand)

**Independent Test**: Trigger multiple concurrent workflows and observe runner pod scaling

### Implementation for User Story 2

- [ ] T021 [US2] Verify runner scale set configuration has minRunners: 0 and maxRunners: 3 in k8s/configs/arc-runners/helmrelease.yaml
- [ ] T022 [US2] Verify resource limits are set (4 CPU, 8GB RAM) in runner template
- [ ] T023 [US2] Test scale-to-zero: Verify no runner pods exist when idle
- [ ] T024 [US2] Test scale-up: Trigger workflow and verify runner pod is created within 2 minutes
- [ ] T025 [US2] Test concurrent scaling: Trigger 2-3 workflows simultaneously and verify multiple runner pods created
- [ ] T026 [US2] Test scale-down: Wait for jobs to complete and verify runners scale down within 5 minutes

**Checkpoint**: User Story 2 complete - runners scale automatically based on demand

---

## Phase 5: User Story 3 - GitOps-Managed Deployment (Priority: P3)

**Goal**: Verify all configuration is managed via Git and FluxCD reconciliation

**Independent Test**: Make a configuration change via Git commit and verify FluxCD applies it

### Implementation for User Story 3

- [ ] T027 [US3] Verify all ARC manifests are committed to Git repository
- [ ] T028 [US3] Verify Flux Kustomization includes SOPS decryption configuration for arc-runners
- [ ] T029 [US3] Test GitOps flow: Change a non-critical config value via Git commit
- [ ] T030 [US3] Verify FluxCD reconciles the change to the cluster automatically
- [ ] T031 [US3] Document the GitOps workflow in quickstart.md if not already documented

**Checkpoint**: User Story 3 complete - all changes managed via GitOps

---

## Phase 6: Polish & Cross-Cutting Concerns

**Purpose**: Documentation and final validation

- [ ] T032 [P] Update docs/ with ARC deployment documentation if needed
- [ ] T033 [P] Verify all secrets are SOPS-encrypted (no plaintext in repository)
- [ ] T034 Run through quickstart.md validation steps
- [ ] T035 Verify success criteria SC-001 through SC-005 from spec.md

---

## Dependencies & Execution Order

### Phase Dependencies

- **Setup (Phase 1)**: No dependencies - can start immediately
- **Foundational (Phase 2)**: Depends on Setup completion - BLOCKS all user stories
- **User Stories (Phase 3-5)**: All depend on Foundational phase completion
  - US1 → US2 → US3 (sequential, each builds on verification of previous)
- **Polish (Phase 6)**: Depends on all user stories being complete

### User Story Dependencies

- **User Story 1 (P1)**: Can start after Foundational (Phase 2) - Core deployment
- **User Story 2 (P2)**: Depends on US1 being deployed - Tests scaling behavior
- **User Story 3 (P3)**: Depends on US1 being deployed - Tests GitOps flow

### Within Each User Story

- Manifest creation before Kustomization updates
- Kustomization updates before FluxCD reconciliation
- Deployment verification before testing

### Parallel Opportunities

- T003, T007, T008 can run in parallel (different files)
- T010-T012 (controller) and T013-T015 (runner) can be developed in parallel
- T032, T033 can run in parallel during Polish phase

---

## Parallel Example: Phase 1 & 2

```bash
# Launch parallel tasks in Setup:
Task: "Create HelmRepository in k8s/infrastructure/actions-runner-controller/helmrepository.yaml"

# Launch parallel tasks in Foundational:
Task: "Create arc-systems namespace in k8s/infrastructure/actions-runner-controller/namespace.yaml"
Task: "Create arc-runners namespace in k8s/configs/arc-runners/namespace.yaml"
```

---

## Implementation Strategy

### MVP First (User Story 1 Only)

1. Complete Phase 1: Setup (directory structure, HelmRepository)
2. Complete Phase 2: Foundational (GitHub App, namespaces, secret)
3. Complete Phase 3: User Story 1 (controller + runner deployment)
4. **STOP and VALIDATE**: Test workflow execution on homelab runner
5. Deploy to production if ready

### Incremental Delivery

1. Complete Setup + Foundational → Foundation ready
2. Add User Story 1 → Test workflow execution → MVP deployed!
3. Add User Story 2 → Verify scaling behavior
4. Add User Story 3 → Verify GitOps management
5. Each story adds confidence without breaking previous functionality

### Single Developer Strategy

1. Follow phases sequentially (1 → 2 → 3 → 4 → 5 → 6)
2. Verify each checkpoint before proceeding
3. Commit after each logical group of tasks

---

## Notes

- [P] tasks = different files, no dependencies
- [Story] label maps task to specific user story for traceability
- User Stories 2 and 3 are primarily verification/testing tasks (scaling and GitOps already configured in US1)
- GitHub App creation (T004-T006) is a manual prerequisite outside of Git
- Verify FluxCD reconciliation after each push
- Stop at any checkpoint to validate story independently
