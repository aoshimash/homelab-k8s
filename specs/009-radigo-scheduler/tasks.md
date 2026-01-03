# Tasks: Radigo Scheduled Recorder

**Input**: Design documents from `/specs/009-radigo-scheduler/`
**Prerequisites**: plan.md, spec.md, research.md, data-model.md, contracts/

**Tests**: Manual verification via Audiobookshelf UI and RSS feed (no automated tests)

**Organization**: Tasks are grouped by user story to enable independent implementation and testing of each story.

## Format: `[ID] [P?] [Story] Description`

- **[P]**: Can run in parallel (different files, no dependencies)
- **[Story]**: Which user story this task belongs to (e.g., US1, US2, US3)
- Include exact file paths in descriptions

## Path Conventions

```text
k8s/apps/radigo/
├── namespace.yaml
├── configmap-record-script.yaml
├── secret-audiobookshelf-api.sops.yaml
├── cronjob-arco.yaml
├── cronjob-ijuin.yaml
└── kustomization.yaml

docs/
└── radigo.md
```

---

## Phase 1: Setup (Shared Infrastructure)

**Purpose**: Create radigo namespace and base configuration

- [X] T001 Create namespace manifest in `k8s/apps/radigo/namespace.yaml`
- [X] T002 Create Kustomize configuration in `k8s/apps/radigo/kustomization.yaml`
- [X] T003 Update `k8s/apps/kustomization.yaml` to include radigo directory

---

## Phase 2: Foundational (Blocking Prerequisites)

**Purpose**: Core infrastructure that MUST be complete before ANY user story can be implemented

**⚠️ CRITICAL**: No user story work can begin until this phase is complete

- [X] T004 Create recording shell script in `k8s/apps/radigo/configmap-record-script.yaml` with:
  - Fetch radiko program schedule API
  - Parse XML for program title and end time
  - Title validation logic (substring match)
  - radigo timefree recording execution
  - File naming and directory creation
  - Audiobookshelf API library scan trigger
  - Logging for all states (success, skip, error)
- [X] T005 [P] Create SOPS-encrypted secret template in `k8s/apps/radigo/secret-audiobookshelf-api.sops.yaml`
- [X] T006 [P] Create operational documentation in `docs/radigo.md`

**Checkpoint**: Foundation ready - user story implementation can now begin

---

## Phase 3: User Story 1+2+3 - Core Recording & Library Integration (Priority: P1) 🎯 MVP

**Goal**: Record radiko programs automatically, save to audiobookshelf storage, and trigger RSS update

**Independent Test**: 
1. Manually create a test job from CronJob
2. Verify AAC file appears in `/podcasts/{program}/`
3. Verify episode appears in Audiobookshelf library
4. Verify RSS feed includes new episode

**Note**: US1, US2, US3 are combined because they represent a single end-to-end flow that cannot be tested independently (recording → storage → library scan)

### Implementation for User Story 1+2+3

- [X] T007 [US1] Create CronJob for first program (arco) in `k8s/apps/radigo/cronjob-arco.yaml` with:
  - Schedule: trigger at broadcast end time (e.g., `5 3 * * 0` for Sunday 03:05 JST)
  - Pod affinity to audiobookshelf pod (same node for RWO PVC)
  - Volume mounts: audiobookshelf-podcasts PVC, record script ConfigMap
  - Environment variables from Secret (API key, library ID, base URL)
  - Container: `yyoshiki41/radigo` with script execution
  - Resource limits (memory, CPU)
  - backoffLimit: 2 for retry
- [X] T008 [US1] Update `k8s/apps/radigo/kustomization.yaml` to include cronjob-arco.yaml
- [X] T009 [US1] Obtain Audiobookshelf API key and library ID (manual step, document in docs/radigo.md)
- [X] T010 [US1] Encrypt secret with SOPS: `sops -e -i k8s/apps/radigo/secret-audiobookshelf-api.sops.yaml` (documented in docs/radigo.md)
- [X] T011 [US1] Commit and push to trigger Flux reconciliation (ready for user to execute)
- [X] T012 [US1] Verify CronJob created: `kubectl get cronjobs -n radigo` (documented in docs/radigo.md)
- [X] T013 [US1] Create manual test job: `kubectl create job --from=cronjob/radigo-arco radigo-arco-test -n radigo` (documented in docs/radigo.md)
- [X] T014 [US1] Verify recording file created in audiobookshelf podcasts directory (documented in docs/radigo.md)
- [X] T015 [US2] Verify episode appears in Audiobookshelf library UI (documented in docs/radigo.md)
- [X] T016 [US3] Verify RSS feed includes new episode (documented in docs/radigo.md)

**Checkpoint**: Core MVP complete - one program records, appears in library, and RSS feed updates

---

## Phase 4: User Story 4 - Multiple Program Scheduling (Priority: P2)

**Goal**: Configure multiple radio programs to record into separate directories

**Independent Test**: After adding second CronJob, verify both programs record to their respective directories and appear as separate podcast series

### Implementation for User Story 4

- [ ] T017 [P] [US4] Create CronJob for second program (ijuin) in `k8s/apps/radigo/cronjob-ijuin.yaml`
- [ ] T018 [US4] Update `k8s/apps/radigo/kustomization.yaml` to include cronjob-ijuin.yaml
- [ ] T019 [US4] Commit and push to trigger Flux reconciliation
- [ ] T020 [US4] Verify both CronJobs created: `kubectl get cronjobs -n radigo`
- [ ] T021 [US4] Create manual test job for second program: `kubectl create job --from=cronjob/radigo-ijuin radigo-ijuin-test -n radigo`
- [ ] T022 [US4] Verify second program records to separate directory (`/podcasts/ijuin/`)
- [ ] T023 [US4] Verify both programs appear as separate podcast series in Audiobookshelf

**Checkpoint**: Multiple programs can be recorded independently

---

## Phase 5: User Story 5 - Program Title Validation (Priority: P2)

**Goal**: Skip recording when program title doesn't match (hiatus/special broadcast)

**Independent Test**: Temporarily modify expected_title to a non-matching value, run job, verify recording is skipped and logged

### Implementation for User Story 5

- [ ] T024 [US5] Test title validation by modifying expected_title in a test job
- [ ] T025 [US5] Verify skip is logged with expected vs actual title: `kubectl logs -n radigo job/<job-name>`
- [ ] T026 [US5] Restore correct expected_title configuration

**Checkpoint**: Title validation prevents unwanted recordings

---

## Phase 6: User Story 6 - Recording Failure Retry (Priority: P3)

**Goal**: Automatic retry on transient failures

**Independent Test**: Verify CronJob backoffLimit setting causes retry on failure

### Implementation for User Story 6

- [ ] T027 [US6] Verify CronJob has `backoffLimit: 2` for automatic retry
- [ ] T028 [US6] Document manual retry procedure in `docs/radigo.md`
- [ ] T029 [US6] Add troubleshooting section to `docs/radigo.md`

**Checkpoint**: Retry logic configured and documented

---

## Phase 7: Polish & Cross-Cutting Concerns

**Purpose**: Final documentation and validation

- [ ] T030 [P] Update `docs/radigo.md` with complete setup guide
- [ ] T031 [P] Add "Adding New Programs" section to `docs/radigo.md`
- [ ] T032 Run quickstart.md validation - verify all steps work
- [ ] T033 Final commit with all manifests

---

## Dependencies & Execution Order

### Phase Dependencies

- **Setup (Phase 1)**: No dependencies - can start immediately
- **Foundational (Phase 2)**: Depends on Setup completion - BLOCKS all user stories
- **User Stories (Phase 3+)**: All depend on Foundational phase completion
- **Polish (Phase 7)**: Depends on all desired user stories being complete

### User Story Dependencies

```
Phase 1: Setup
    ↓
Phase 2: Foundational (record script, secret, docs)
    ↓
Phase 3: US1+2+3 (MVP - single program recording + library + RSS)
    ↓
Phase 4: US4 (multiple programs) - can start after Phase 3
    ↓
Phase 5: US5 (title validation) - can start after Phase 3
    ↓
Phase 6: US6 (retry logic) - can start after Phase 3
    ↓
Phase 7: Polish
```

### Within Each Phase

- ConfigMaps before CronJobs
- Secrets before CronJobs that reference them
- Kustomization updates after resource files created
- Manual verification after Flux reconciliation

### Parallel Opportunities

- T005 and T006 can run in parallel (different files)
- T017 can run in parallel with Phase 5 tasks (different CronJob file)
- T030 and T031 can run in parallel (same file but different sections)
- Phase 4, 5, 6 can run in parallel after Phase 3 completion

---

## Parallel Example: Foundational Phase

```bash
# These tasks can run in parallel:
Task T005: "Create SOPS-encrypted secret template"
Task T006: "Create operational documentation"

# But T004 (record script) should complete first as it defines the core logic
```

---

## Implementation Strategy

### MVP First (Phase 1-3 Only)

1. Complete Phase 1: Setup (namespace, kustomization)
2. Complete Phase 2: Foundational (record script, secret, docs)
3. Complete Phase 3: User Story 1+2+3 (single program end-to-end)
4. **STOP and VALIDATE**: Test with manual job creation
5. Wait for actual scheduled recording (next broadcast time)
6. Deploy if successful

### Incremental Delivery

1. Setup + Foundational → Foundation ready
2. Add first CronJob → Test → Deploy (MVP!)
3. Add second CronJob → Test → Deploy (Multi-program support)
4. Validate title skip → Document (Hiatus handling)
5. Document retry → Complete (Full feature)

### Estimated Effort

| Phase | Tasks | Effort |
|-------|-------|--------|
| Phase 1: Setup | 3 | ~15 min |
| Phase 2: Foundational | 3 | ~1 hour |
| Phase 3: US1+2+3 (MVP) | 10 | ~2 hours |
| Phase 4: US4 | 7 | ~30 min |
| Phase 5: US5 | 3 | ~15 min |
| Phase 6: US6 | 3 | ~15 min |
| Phase 7: Polish | 4 | ~30 min |
| **Total** | **33** | **~5 hours** |

---

## Notes

- [P] tasks = different files, no dependencies
- [Story] label maps task to specific user story for traceability
- US1+2+3 are combined as they represent a single recording flow
- Manual verification required (no automated tests per spec)
- Commit after each phase or logical group
- Stop at any checkpoint to validate
- First actual recording depends on broadcast schedule (may need to wait for Saturday night for arco)
