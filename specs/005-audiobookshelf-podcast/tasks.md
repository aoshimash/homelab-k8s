# Tasks: Audiobookshelf Podcast Server

**Input**: Design documents from `/specs/005-audiobookshelf-podcast/`
**Prerequisites**: plan.md, spec.md, research.md, data-model.md, contracts/reconciliation.md, quickstart.md

**Organization**: Tasks are grouped by user story to enable independent implementation and testing of each story.

## Format: `[ID] [P?] [Story] Description`

- **[P]**: Can run in parallel (different files, no dependencies)
- **[Story]**: Which user story this task belongs to (e.g., US1, US2, US3)
- Include exact file paths in descriptions

## Path Conventions

- **Kubernetes manifests**: `k8s/apps/audiobookshelf/`
- **Flux configuration**: `k8s/flux/`
- **Documentation**: `docs/`

---

## Phase 1: Setup (Directory Structure)

**Purpose**: Create directory structure and Flux Kustomization for apps

- [x] T001 Create apps directory structure at `k8s/apps/`
- [x] T002 Create Flux Kustomization for apps at `k8s/flux/apps-kustomization.yaml`
- [x] T003 Create apps root kustomization at `k8s/apps/kustomization.yaml`
- [x] T004 Update `k8s/flux/kustomization.yaml` to include apps-kustomization.yaml

---

## Phase 2: Foundational (Kubernetes Resources)

**Purpose**: Core Kubernetes resources that MUST be complete before application can run

**⚠️ CRITICAL**: Application deployment depends on all these resources

- [x] T005 [P] Create namespace manifest at `k8s/apps/audiobookshelf/namespace.yaml`
- [x] T006 [P] Create PVC manifest (config + podcasts) at `k8s/apps/audiobookshelf/pvc.yaml`
- [x] T007 [P] Create Audiobookshelf kustomization at `k8s/apps/audiobookshelf/kustomization.yaml`

**Checkpoint**: Foundational resources ready - deployment can now be created

---

## Phase 3: User Story 1 - Web Interface Access (Priority: P1) 🎯 MVP

**Goal**: Deploy Audiobookshelf and make web UI accessible via Tailscale Ingress

**Independent Test**: Access `https://audiobookshelf.<tailnet>.ts.net` from a tailnet device and see the Audiobookshelf login page

### Implementation for User Story 1

- [x] T008 [US1] Create Deployment manifest at `k8s/apps/audiobookshelf/deployment.yaml`
- [x] T009 [US1] Create Service manifest at `k8s/apps/audiobookshelf/service.yaml`
- [x] T010 [US1] Create Tailscale Ingress manifest at `k8s/apps/audiobookshelf/ingress.yaml`
- [ ] T011 [US1] Commit manifests and trigger Flux reconciliation
- [ ] T012 [US1] Verify pod is running: `kubectl get pods -n audiobookshelf`
- [ ] T013 [US1] Verify Ingress is synced: `kubectl get ingress -n audiobookshelf`
- [ ] T014 [US1] Access web UI from tailnet device and create admin account

**Checkpoint**: Web UI accessible via Tailscale - User Story 1 complete

---

## Phase 4: User Story 2 - RSS Subscription (Priority: P1)

**Goal**: Verify podcast RSS feed subscription works correctly

**Independent Test**: Add a podcast RSS feed and verify episodes are listed and begin downloading

### Implementation for User Story 2

- [ ] T015 [US2] Configure podcast library in Audiobookshelf UI (Settings → Libraries → Add `/podcasts`)
- [ ] T016 [US2] Add a test podcast RSS feed via web UI
- [ ] T017 [US2] Verify episodes are listed with metadata (title, description, duration)
- [ ] T018 [US2] Verify episode download starts and completes successfully

**Checkpoint**: RSS subscription working - User Story 2 complete

---

## Phase 5: User Story 3 - Persistent Storage (Priority: P1)

**Goal**: Verify data persists across pod restarts

**Independent Test**: Restart pod and confirm all data (podcasts, users, progress) survives

### Implementation for User Story 3

- [ ] T019 [US3] Verify PVCs are bound: `kubectl get pvc -n audiobookshelf`
- [ ] T020 [US3] Note current library state (podcasts, episodes, user accounts)
- [ ] T021 [US3] Restart pod: `kubectl rollout restart deployment/audiobookshelf -n audiobookshelf`
- [ ] T022 [US3] Verify pod restarts successfully and reaches Ready state
- [ ] T023 [US3] Verify all data persisted (library contents, user accounts, listening progress)

**Checkpoint**: Data persistence verified - User Story 3 complete

---

## Phase 6: User Story 4 - GitOps Management (Priority: P2)

**Goal**: Verify Flux reconciliation works correctly

**Independent Test**: Make a configuration change in Git and verify Flux applies it automatically

### Implementation for User Story 4

- [ ] T024 [US4] Verify Flux Kustomization status: `flux get kustomizations apps`
- [ ] T025 [US4] Make a minor change (e.g., add label to Deployment)
- [ ] T026 [US4] Commit change to Git and push
- [ ] T027 [US4] Force reconciliation: `flux reconcile kustomization apps --with-source`
- [ ] T028 [US4] Verify change is applied to cluster

**Checkpoint**: GitOps workflow verified - User Story 4 complete

---

## Phase 7: User Story 5 - Secure Access (Priority: P2)

**Goal**: Verify service is only accessible from tailnet

**Independent Test**: Confirm access works from tailnet but fails from public internet

### Implementation for User Story 5

- [ ] T029 [US5] Verify TLS is automatically provisioned (HTTPS works)
- [ ] T030 [US5] Verify access from tailnet device succeeds
- [ ] T031 [US5] Verify access from outside tailnet fails (timeout or connection refused)
- [ ] T032 [US5] Check Tailscale admin console for `audiobookshelf` device entry

**Checkpoint**: Secure tailnet-only access verified - User Story 5 complete

---

## Phase 8: Polish & Documentation

**Purpose**: Final documentation and verification

- [x] T033 [P] Create operational documentation at `docs/audiobookshelf.md`
- [ ] T034 [P] Update `docs/` index if applicable
- [ ] T035 Run quickstart.md verification checklist
- [ ] T036 Mark spec tasks.md as complete

---

## Dependencies & Execution Order

### Phase Dependencies

- **Setup (Phase 1)**: No dependencies - can start immediately
- **Foundational (Phase 2)**: Depends on Setup completion - BLOCKS deployment
- **User Story 1 (Phase 3)**: Depends on Foundational - Core deployment
- **User Story 2 (Phase 4)**: Depends on User Story 1 - Requires running app
- **User Story 3 (Phase 5)**: Depends on User Story 2 - Needs data to persist
- **User Story 4 (Phase 6)**: Depends on User Story 1 - Requires deployed app
- **User Story 5 (Phase 7)**: Depends on User Story 1 - Requires Ingress
- **Polish (Phase 8)**: Depends on all user stories being verified

### User Story Dependencies

| Story | Depends On | Can Run In Parallel With |
|-------|------------|--------------------------|
| US1 | Foundational | None (MVP, must complete first) |
| US2 | US1 | US4, US5 |
| US3 | US2 | US4, US5 |
| US4 | US1 | US2, US3, US5 |
| US5 | US1 | US2, US3, US4 |

### Within Each Phase

- Tasks marked [P] can run in parallel (different files)
- Tasks without [P] must run sequentially

### Parallel Opportunities

**Phase 2 (all can run in parallel):**
```
T005: namespace.yaml
T006: pvc.yaml
T007: kustomization.yaml
```

**Phase 8 (can run in parallel):**
```
T033: docs/audiobookshelf.md
T034: docs/ index update
```

---

## Implementation Strategy

### MVP First (User Stories 1-3)

1. Complete Phase 1: Setup
2. Complete Phase 2: Foundational
3. Complete Phase 3: User Story 1 (Web UI accessible)
4. **STOP and VALIDATE**: Verify web UI works
5. Complete Phase 4: User Story 2 (RSS works)
6. Complete Phase 5: User Story 3 (Data persists)
7. **MVP COMPLETE**: Core functionality verified

### Full Delivery

8. Complete Phase 6: User Story 4 (GitOps verified)
9. Complete Phase 7: User Story 5 (Security verified)
10. Complete Phase 8: Documentation

### Estimated Task Counts

| Phase | Tasks | Parallelizable |
|-------|-------|----------------|
| Setup | 4 | 0 |
| Foundational | 3 | 3 |
| US1 (Web UI) | 7 | 0 |
| US2 (RSS) | 4 | 0 |
| US3 (Persistence) | 5 | 0 |
| US4 (GitOps) | 5 | 0 |
| US5 (Security) | 4 | 0 |
| Polish | 4 | 2 |
| **Total** | **36** | **5** |

---

## Notes

- All manifests should be based on templates in `data-model.md`
- Verification steps follow `quickstart.md` procedures
- Error scenarios documented in `contracts/reconciliation.md`
- No SOPS secrets required for this deployment (user accounts managed in-app)
- Commit after each logical group of tasks
- Stop at any checkpoint to validate independently
