# Tasks: Home Assistant導入とSwitchBot Hub 2のMatter連携

**Input**: Design documents from `/specs/011-home-assistant/`
**Prerequisites**: plan.md, spec.md, research.md, data-model.md, contracts/

**Tests**: Manual verification via kubectl and Home Assistant UI (no automated tests required)

**Organization**: Tasks are grouped by user story to enable independent implementation and testing of each story.

## Format: `[ID] [P?] [Story] Description`

- **[P]**: Can run in parallel (different files, no dependencies)
- **[Story]**: Which user story this task belongs to (e.g., US1, US2, US3)
- Include exact file paths in descriptions

## Path Conventions

- **Kubernetes manifests**: `k8s/apps/home-assistant/`
- **Documentation**: `docs/`
- Follows existing `audiobookshelf` pattern in the codebase

---

## Phase 1: Setup (Directory Structure)

**Purpose**: Create directory structure and base kustomization files

- [x] T001 Create directory structure `k8s/apps/home-assistant/app/`
- [x] T002 [P] Create namespace manifest in `k8s/apps/home-assistant/namespace.yaml`
- [x] T003 [P] Create root kustomization in `k8s/apps/home-assistant/kustomization.yaml`
- [x] T004 Update apps kustomization to include home-assistant in `k8s/apps/kustomization.yaml`

**Checkpoint**: Directory structure ready for resource manifests

---

## Phase 2: Foundational (Storage)

**Purpose**: Create PVC that MUST be available before Deployment

**⚠️ CRITICAL**: PVC must be created and bound before pod can start

- [ ] T005 Create PVC manifest (5Gi Longhorn) in `k8s/apps/home-assistant/app/pvc.yaml`

**Checkpoint**: Storage foundation ready - user story implementation can begin

---

## Phase 3: User Story 1 - Home AssistantへのWebアクセス (Priority: P1) 🎯 MVP

**Goal**: Deploy Home Assistant and enable external access via Tailscale Ingress

**Independent Test**: Access `https://home-assistant.<tailnet>.ts.net` and complete onboarding

### Implementation for User Story 1

- [ ] T006 [P] [US1] Create Deployment manifest with hostNetwork in `k8s/apps/home-assistant/app/deployment.yaml`
- [ ] T007 [P] [US1] Create Service manifest (port 80 → 8123) in `k8s/apps/home-assistant/app/service.yaml`
- [ ] T008 [P] [US1] Create Tailscale Ingress manifest in `k8s/apps/home-assistant/app/ingress.yaml`
- [ ] T009 [US1] Create app-level kustomization in `k8s/apps/home-assistant/app/kustomization.yaml`
- [ ] T010 [US1] Verify Flux reconciliation and pod status with `kubectl get all -n home-assistant`
- [ ] T011 [US1] Verify Tailscale Ingress access via `https://home-assistant.<tailnet>.ts.net`
- [ ] T012 [US1] Complete Home Assistant onboarding and verify persistence after pod restart

**Checkpoint**: User Story 1 complete - Home Assistant accessible via Tailscale, data persisted

---

## Phase 4: User Story 2 - SwitchBot Hub 2とのMatter接続 (Priority: P2)

**Goal**: Enable Matter integration and pair SwitchBot Hub 2

**Independent Test**: SwitchBot Hub 2 appears in Home Assistant devices and can be controlled

**Prerequisites**: User Story 1 complete (Home Assistant running with hostNetwork)

### Implementation for User Story 2

- [ ] T013 [US2] Verify hostNetwork is active and mDNS is working on the node
- [ ] T014 [US2] Enable Matter integration in Home Assistant UI (Settings → Devices & Services → Add Integration → Matter)
- [ ] T015 [US2] Prepare SwitchBot Hub 2 Matter pairing code via SwitchBot app
- [ ] T016 [US2] Pair SwitchBot Hub 2 with Home Assistant via Matter
- [ ] T017 [US2] Verify device control response time (< 1 second)
- [ ] T018 [US2] Verify Matter credentials persist after pod restart

**Checkpoint**: User Story 2 complete - Matter devices operational

---

## Phase 5: User Story 3 - YAMLによるオートメーション管理 (Priority: P3)

**Goal**: Enable Git-managed automations via ConfigMap

**Independent Test**: ConfigMap automation executes on Home Assistant startup

**Prerequisites**: User Story 1 complete (Home Assistant running)

### Implementation for User Story 3

- [ ] T019 [P] [US3] Create ConfigMap with sample automation in `k8s/apps/home-assistant/app/configmap.yaml`
- [ ] T020 [US3] Update Deployment to mount ConfigMap at `/config/automations.yaml` in `k8s/apps/home-assistant/app/deployment.yaml`
- [ ] T021 [US3] Update app kustomization to include ConfigMap in `k8s/apps/home-assistant/app/kustomization.yaml`
- [ ] T022 [US3] Verify automation is loaded in Home Assistant (Settings → Automations)
- [ ] T023 [US3] Test automation execution (startup notification should appear)
- [ ] T024 [US3] Test ConfigMap update workflow (modify, apply, reload)

**Checkpoint**: User Story 3 complete - Automations managed via Git

---

## Phase 6: Polish & Documentation

**Purpose**: Documentation and final validation

- [ ] T025 [P] Create operational documentation in `docs/home-assistant.md`
- [ ] T026 [P] Update README if needed to reference Home Assistant
- [ ] T027 Run quickstart.md validation steps
- [ ] T028 Final verification: all success criteria from spec.md

---

## Dependencies & Execution Order

### Phase Dependencies

- **Setup (Phase 1)**: No dependencies - can start immediately
- **Foundational (Phase 2)**: Depends on Setup completion - BLOCKS all user stories
- **User Story 1 (Phase 3)**: Depends on Foundational (PVC must exist)
- **User Story 2 (Phase 4)**: Depends on User Story 1 (Home Assistant must be running with hostNetwork)
- **User Story 3 (Phase 5)**: Depends on User Story 1 (Home Assistant must be running)
- **Polish (Phase 6)**: Depends on all user stories being complete

### User Story Dependencies

```
Phase 1: Setup
    │
    ▼
Phase 2: Foundational (PVC)
    │
    ▼
Phase 3: User Story 1 (P1) ─── MVP COMPLETE
    │
    ├───────────────────┐
    ▼                   ▼
Phase 4: US2 (P2)   Phase 5: US3 (P3)
    │                   │
    └───────┬───────────┘
            ▼
    Phase 6: Polish
```

- **User Story 1 (P1)**: Can start after Foundational - No dependencies on other stories
- **User Story 2 (P2)**: Requires User Story 1 complete (hostNetwork active)
- **User Story 3 (P3)**: Requires User Story 1 complete (can run in parallel with US2)

### Parallel Opportunities

**Within Phase 1 (Setup)**:
- T002, T003 can run in parallel (different files)

**Within Phase 3 (User Story 1)**:
- T006, T007, T008 can run in parallel (different manifest files)

**Within Phase 5 (User Story 3)**:
- T019 can run in parallel with other US3 tasks initially

**Cross-Story Parallelism**:
- After US1 complete: US2 and US3 can proceed in parallel

---

## Parallel Example: User Story 1 Manifests

```bash
# Launch all manifest creation tasks together:
Task T006: "Create Deployment manifest in k8s/apps/home-assistant/app/deployment.yaml"
Task T007: "Create Service manifest in k8s/apps/home-assistant/app/service.yaml"
Task T008: "Create Ingress manifest in k8s/apps/home-assistant/app/ingress.yaml"
```

---

## Implementation Strategy

### MVP First (User Story 1 Only)

1. Complete Phase 1: Setup (T001-T004)
2. Complete Phase 2: Foundational (T005)
3. Complete Phase 3: User Story 1 (T006-T012)
4. **STOP and VALIDATE**: Access Home Assistant via Tailscale
5. Deploy/demo if ready - **MVP COMPLETE**

### Incremental Delivery

1. Setup + Foundational → Foundation ready
2. Add User Story 1 → Test independently → **MVP Ready**
3. Add User Story 2 → Test Matter integration → Enhanced functionality
4. Add User Story 3 → Test automations → Full feature complete
5. Polish → Documentation complete

### Single Developer Strategy

Execute in strict order:
1. T001 → T002 → T003 → T004 (Setup)
2. T005 (Foundational)
3. T006 → T007 → T008 → T009 → T010 → T011 → T012 (US1)
4. T013 → T014 → T015 → T016 → T017 → T018 (US2)
5. T019 → T020 → T021 → T022 → T023 → T024 (US3)
6. T025 → T026 → T027 → T028 (Polish)

---

## Notes

- [P] tasks = different files, no dependencies
- [Story] label maps task to specific user story for traceability
- Each user story should be independently completable and testable
- Commit after each phase or logical group
- Stop at any checkpoint to validate story independently
- US2 requires physical SwitchBot Hub 2 device for testing
- US3 can be tested without Matter devices
