# Tasks: Manage Cilium with Flux

**Input**: Design documents from `/specs/001-cilium-flux/`
**Prerequisites**: plan.md (required), spec.md (required for user stories), research.md, data-model.md, contracts/

**Tests**: Tests are not explicitly requested in the feature specification, so no test tasks are included.

**Organization**: Tasks are grouped by user story to enable independent implementation and testing of each story.

## Format: `[ID] [P?] [Story] Description`

- **[P]**: Can run in parallel (different files, no dependencies)
- **[Story]**: Which user story this task belongs to (e.g., US1, US2, US3)
- Include exact file paths in descriptions

## Path Conventions

- Infrastructure manifests: `k8s/infrastructure/`
- Flux Kustomizations: `k8s/flux/infrastructure-kustomization.yaml` (and future siblings)
- Documentation: `docs/`

---

## Phase 1: Setup (Shared Infrastructure)

**Purpose**: Create directory structure for Flux-managed infrastructure

- [x] T001 Create directory structure `k8s/infrastructure/cilium/` for Cilium manifests
- [x] T002 Create `k8s/flux/infrastructure-kustomization.yaml` for Flux-managed infrastructure reconciliation

---

## Phase 2: Foundational (Blocking Prerequisites)

**Purpose**: Core infrastructure that MUST be complete before ANY user story can be implemented

**⚠️ CRITICAL**: No user story work can begin until this phase is complete

**Note**: For this GitOps infrastructure feature, there are no blocking foundational tasks beyond directory structure (Phase 1). The Flux controllers are already installed and reconciling.

**Checkpoint**: Foundation ready - user story implementation can now begin

---

## Phase 3: User Story 1 - Declarative Cilium installation (Priority: P1) 🎯 MVP

**Goal**: Cilium is installed and updated via Flux (GitOps) so that the cluster networking stack is reproducible, auditable, and recoverable without manual Helm actions.

**Independent Test**: A fresh environment/cluster (or a reset environment) can be brought to a working networking state by applying the Git repository state and letting Flux reconcile, without running any manual Helm install/upgrade commands.

### Implementation for User Story 1

- [x] T003 [P] [US1] Create HelmRepository resource for Cilium chart source in `k8s/infrastructure/cilium/helmrepository.yaml`
- [x] T004 [US1] Create HelmRelease resource for Cilium in `k8s/infrastructure/cilium/helmrelease.yaml` with pinned version matching current manual installation
  - **NOTE**: Values are based on documentation and are provisional. T009/T010 must verify and update values from the actual running cluster before deployment to ensure zero-downtime migration.
- [x] T005 [US1] Create kustomization.yaml in `k8s/infrastructure/cilium/kustomization.yaml` to include HelmRepository and HelmRelease
- [x] T006 [US1] Create Flux Kustomization resource in `k8s/flux/infrastructure-kustomization.yaml` to reconcile `k8s/infrastructure/` path
- [x] T007 [US1] (Removed) Extra Kustomize wrapper is not needed; reference the Flux Kustomization YAML directly from `k8s/flux/kustomization.yaml`
- [x] T008 [US1] Update `k8s/flux/kustomization.yaml` to include `k8s/flux/infrastructure-kustomization.yaml` in resources

**Checkpoint**: At this point, User Story 1 should be fully functional and testable independently. Flux should reconcile the HelmRepository and HelmRelease, and Cilium should be managed declaratively.

---

## Phase 4: User Story 2 - Safe transition from manual management (Priority: P2)

**Goal**: The migration avoids networking downtime and prevents "double-management" conflicts, so that workloads keep running during the transition.

**Independent Test**: During the cutover window, there is at most one active source of truth managing Cilium, and Cilium remains ready throughout.

### Implementation for User Story 2

- [ ] T009 [US2] **CRITICAL**: Capture current Helm release details (version and values) from running cluster using `helm get values cilium -n kube-system` and `helm list -n kube-system`
  - **REQUIRED BEFORE DEPLOYMENT**: This task must be completed to verify the HelmRelease values match the actual cluster state, preventing version mismatches or configuration drift that could cause networking disruption.
- [ ] T010 [US2] **CRITICAL**: Encode captured Helm values into HelmRelease spec.values in `k8s/infrastructure/cilium/helmrelease.yaml` to match current configuration
  - **REQUIRED BEFORE DEPLOYMENT**: Update the HelmRelease with verified values from T009 to ensure zero-downtime migration.
- [ ] T011 [US2] Set HelmRelease spec.releaseName to match existing manual Helm release name (`cilium`) in `k8s/infrastructure/cilium/helmrelease.yaml`
- [ ] T012 [US2] Verify HelmRelease adopts existing release by checking Flux reconciliation status: `kubectl -n kube-system get helmreleases cilium`
- [ ] T013 [US2] Verify Cilium pods remain Ready during cutover: `kubectl -n kube-system get pods -l app.kubernetes.io/part-of=cilium`
- [ ] T014 [US2] Verify nodes remain Ready during cutover: `kubectl get nodes`
- [ ] T015 [US2] Verify no double management by confirming manual Helm release is adopted: `helm list -n kube-system` should show release managed by Flux

**Checkpoint**: At this point, User Stories 1 AND 2 should both work independently. Migration is complete with zero downtime, and Flux is the single source of truth.

---

## Phase 5: User Story 3 - Predictable upgrades and rollback (Priority: P3)

**Goal**: Cilium version/config changes are done via pull requests only (no automatic upgrades), with an easy rollback path, so that changes are reviewable and reversable.

**Independent Test**: A version/config change can be introduced via a PR, observed in the cluster after reconciliation, and reverted by reverting the PR.

### Implementation for User Story 3

- [ ] T016 [US3] Verify HelmRelease has chart version pinned (no automatic upgrades) in `k8s/infrastructure/cilium/helmrelease.yaml`
- [ ] T017 [US3] Document rollback procedure: revert Git commit and verify Flux reconciles back to previous state in `docs/cilium-rollback.md`
- [ ] T018 [US3] Test rollback workflow: make a test change to HelmRelease values, commit, verify reconciliation, then revert commit and verify rollback reconciliation completes within 15 minutes

**Checkpoint**: All user stories should now be independently functional. Rollback capability is validated and documented.

---

## Phase 6: Polish & Cross-Cutting Concerns

**Purpose**: Documentation updates and validation

- [x] T019 [P] Update `docs/talos-bootstrap.md` to remove manual Helm install steps (sections "Post-Bootstrap: Install Cilium CNI" and related)
- [x] T020 [P] Add Flux-managed Cilium installation workflow to `docs/talos-bootstrap.md` referencing the GitOps path
- [x] T021 [P] Add verification commands from quickstart.md to `docs/talos-bootstrap.md` for post-bootstrap validation
- [ ] T022 Run quickstart.md validation: execute all validation commands and confirm Cilium is Flux-managed and healthy

---

## Dependencies & Execution Order

### Phase Dependencies

- **Setup (Phase 1)**: No dependencies - can start immediately
- **Foundational (Phase 2)**: Depends on Setup completion - No blocking tasks for this feature
- **User Stories (Phase 3+)**: All depend on Setup phase completion
  - User stories proceed sequentially in priority order (P1 → P2 → P3)
  - US2 depends on US1 completion (migration builds on declarative setup)
  - US3 depends on US2 completion (rollback validation requires completed migration)
- **Polish (Final Phase)**: Depends on all user stories being complete

### User Story Dependencies

- **User Story 1 (P1)**: Can start after Setup (Phase 1) - No dependencies on other stories
- **User Story 2 (P2)**: Depends on US1 completion - Migration procedure requires HelmRelease to exist
- **User Story 3 (P3)**: Depends on US2 completion - Rollback validation requires completed migration

### Within Each User Story

- Resources before reconciliation verification
- Core implementation before integration checks
- Story complete before moving to next priority

### Parallel Opportunities

- Phase 1 tasks T001 and T002 can run in parallel (different directories)
- Phase 3 tasks T003 and T004 can run in parallel (different files)
- Phase 6 documentation tasks T019, T020, T021 can run in parallel (different sections of same file, but can be worked on sequentially)

---

## Parallel Example: User Story 1

```bash
# Launch HelmRepository and HelmRelease creation together:
Task: "Create HelmRepository resource for Cilium chart source in k8s/infrastructure/cilium/helmrepository.yaml"
Task: "Create HelmRelease resource for Cilium in k8s/infrastructure/cilium/helmrelease.yaml"
```

---

## Implementation Strategy

### MVP First (User Story 1 Only)

1. Complete Phase 1: Setup
2. Complete Phase 3: User Story 1
3. **STOP and VALIDATE**: Verify Flux reconciles Cilium declaratively without manual Helm commands
4. Deploy/demo if ready

### Incremental Delivery

1. Complete Setup → Foundation ready
2. Add User Story 1 → Test independently → Deploy/Demo (MVP!)
3. Add User Story 2 → Test independently → Deploy/Demo (Migration complete)
4. Add User Story 3 → Test independently → Deploy/Demo (Rollback validated)
5. Add Polish → Documentation complete
6. Each story adds value without breaking previous stories

### Sequential Strategy (Recommended for this feature)

With a single operator:

1. Complete Setup
2. Complete User Story 1 → Verify declarative installation works
3. Complete User Story 2 → Verify migration is safe (zero downtime)
4. Complete User Story 3 → Verify rollback capability
5. Complete Polish → Update documentation

**Rationale**: Migration and rollback tasks build on the declarative setup, so sequential execution reduces risk.

---

## Notes

- [P] tasks = different files, no dependencies
- [Story] label maps task to specific user story for traceability
- Each user story should be independently completable and testable
- Commit after each task or logical group
- Stop at any checkpoint to validate story independently
- Avoid: vague tasks, same file conflicts, cross-story dependencies that break independence
- Migration tasks (US2) require access to running cluster to capture current Helm state
- Rollback validation (US3) requires making and reverting test changes
