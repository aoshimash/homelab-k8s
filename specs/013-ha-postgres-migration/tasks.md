# Tasks: Migrate Home Assistant DB to PostgreSQL Cluster

**Input**: Design documents from `/specs/013-ha-postgres-migration/`
**Prerequisites**: plan.md, spec.md, research.md, data-model.md, contracts/

**Organization**: Tasks are grouped by user story to enable independent implementation and testing of each story.

## Format: `[ID] [P?] [Story] Description`

- **[P]**: Can run in parallel (different files, no dependencies)
- **[Story]**: Which user story this task belongs to (e.g., US1, US2, US3)
- Include exact file paths in descriptions

---

## Phase 1: Setup (Shared Infrastructure)

**Purpose**: Verify existing project structure and prepare for migration

- [ ] T001 Verify existing Home Assistant deployment structure in `k8s/apps/home-assistant/`
- [ ] T002 Verify PostgreSQL cluster `postgres-cluster` is Ready in namespace `postgres`
- [ ] T003 [P] Verify Flux Kustomization reconciliation is working for `k8s/apps/`

---

## Phase 2: Foundational (Blocking Prerequisites)

**Purpose**: Core infrastructure that MUST be complete before ANY user story can be implemented

**⚠️ CRITICAL**: No user story work can begin until this phase is complete

- [ ] T004 Create PostgreSQL database `homeassistant` and user `homeassistant` in `postgres-cluster` (operational step; document in runbook)
- [ ] T005 [P] Create SOPS-encrypted Secret `secret-db-url.sops.yaml` in `k8s/apps/home-assistant/app/` with `DB_URL` connection string
- [ ] T006 [P] Create Home Assistant Recorder configuration file `configuration.yaml` in `k8s/apps/home-assistant/app/` with `recorder.db_url: !env_var DB_URL` and retry settings
- [ ] T007 Update `kustomization.yaml` in `k8s/apps/home-assistant/app/` to include `secret-db-url.sops.yaml` and `configuration.yaml`

**Checkpoint**: Foundation ready - user story implementation can now begin

---

## Phase 3: User Story 1 - Preserve history after migration (Priority: P1) 🎯 MVP

**Goal**: Migrate Home Assistant's Recorder database from SQLite to PostgreSQL cluster, preserving all existing history and ensuring new events continue to be recorded.

**Independent Test**: Migrate a non-production copy (or a snapshot) and verify that pre-migration history is visible in Home Assistant UI charts/logbook, and that new events continue to be recorded after migration.

### Implementation for User Story 1

- [ ] T008 [US1] Update `deployment.yaml` in `k8s/apps/home-assistant/app/` to inject `DB_URL` environment variable from Secret `home-assistant-db-url`
- [ ] T009 [US1] Mount `configuration.yaml` ConfigMap into Home Assistant pod at `/config/configuration.yaml` in `deployment.yaml`
- [ ] T010 [US1] Create migration Job manifest `job-db-migrate.yaml` in `k8s/apps/home-assistant/app/` using pgloader to copy SQLite DB to PostgreSQL
- [ ] T011 [US1] Add `job-db-migrate.yaml` to `kustomization.yaml` in `k8s/apps/home-assistant/app/` (temporary; will be removed after migration)
- [ ] T012 [US1] Create pre-cutover backup runbook documenting Longhorn snapshot and CloudNativePG backup procedures in `specs/013-ha-postgres-migration/`
- [ ] T013 [US1] Execute cutover sequence: stop Home Assistant, run migration Job, start Home Assistant with PostgreSQL config
- [ ] T014 [US1] Verify history continuity: check Home Assistant UI shows pre-migration charts/logbook entries and new events are recorded

**Checkpoint**: At this point, User Story 1 should be fully functional - Home Assistant is using PostgreSQL and history is preserved

---

## Phase 4: User Story 2 - Safe rollback on migration failure (Priority: P2)

**Goal**: Provide a clear rollback path so that if the migration fails, service can be restored quickly without permanently losing the original data.

**Independent Test**: Intentionally fail the migration in a test environment and confirm Home Assistant can be restored to the pre-migration state within ≤ 30 minutes.

### Implementation for User Story 2

- [ ] T015 [US2] Document rollback procedure in `specs/013-ha-postgres-migration/contracts/cutover.md` (Git revert steps and SQLite restoration)
- [ ] T016 [US2] Verify pre-migration SQLite database remains accessible in PVC `home-assistant-config` for ≥ 24 hours after cutover
- [ ] T017 [US2] Test rollback procedure: revert Git commits, verify Home Assistant starts with SQLite and core features work (dashboards, automations, device state updates)
- [ ] T018 [US2] Document rollback verification checklist: Home Assistant starts successfully, no user-visible data storage errors for 1 hour after rollback

**Checkpoint**: At this point, User Stories 1 AND 2 should both work independently - migration is complete with rollback capability

---

## Phase 5: User Story 3 - Restore from backup and verify data integrity (Priority: P3)

**Goal**: Validate that Home Assistant database can be restored from backup, proving disaster recovery capability.

**Independent Test**: Restore a backup into a test Home Assistant instance and confirm expected data (history/state) is present.

### Implementation for User Story 3

- [ ] T019 [US3] Document backup validation procedure: restore PostgreSQL backup to test cluster and verify data integrity in `specs/013-ha-postgres-migration/contracts/backup-restore.md`
- [ ] T020 [US3] Execute test restore: restore CloudNativePG backup to a test PostgreSQL cluster and verify Home Assistant can connect
- [ ] T021 [US3] Verify restored data: check historical data is visible and consistent with expectations using Home Assistant UI and basic SQL queries
- [ ] T022 [US3] Document restore verification checklist: historical data visible, row counts match expectations, recent timestamps present

**Checkpoint**: All user stories should now be independently functional - migration, rollback, and backup/restore are all validated

---

## Phase 6: Polish & Cross-Cutting Concerns

**Purpose**: Cleanup and documentation improvements

- [ ] T023 [P] Remove temporary migration Job manifest `job-db-migrate.yaml` from `k8s/apps/home-assistant/app/kustomization.yaml` after 24h observation period
- [ ] T024 [P] Update operational documentation in `docs/` if needed to reflect PostgreSQL usage
- [ ] T025 [P] Verify all SOPS-encrypted secrets are properly encrypted before committing
- [ ] T026 Run quickstart.md validation: verify all steps can be executed successfully
- [ ] T027 Document any operational learnings or gotchas in `specs/013-ha-postgres-migration/`

---

## Dependencies & Execution Order

### Phase Dependencies

- **Setup (Phase 1)**: No dependencies - can start immediately
- **Foundational (Phase 2)**: Depends on Setup completion - BLOCKS all user stories
- **User Stories (Phase 3+)**: All depend on Foundational phase completion
  - User stories can then proceed sequentially in priority order (P1 → P2 → P3)
  - US2 and US3 can be worked on in parallel after US1 is complete
- **Polish (Final Phase)**: Depends on all desired user stories being complete

### User Story Dependencies

- **User Story 1 (P1)**: Can start after Foundational (Phase 2) - No dependencies on other stories
- **User Story 2 (P2)**: Can start after Foundational (Phase 2) - Depends on US1 completion for rollback testing
- **User Story 3 (P3)**: Can start after Foundational (Phase 2) - Can run in parallel with US2 after US1 is complete

### Within Each User Story

- Core infrastructure (Secret, ConfigMap, Deployment updates) before migration execution
- Migration execution before verification
- Verification before moving to next story

### Parallel Opportunities

- All Setup tasks marked [P] can run in parallel
- Foundational tasks T005 and T006 marked [P] can run in parallel (different files)
- Polish tasks marked [P] can run in parallel (different files)
- US2 and US3 can be worked on in parallel after US1 is complete

---

## Parallel Example: Foundational Phase

```bash
# Launch foundational tasks in parallel:
Task: "Create SOPS-encrypted Secret secret-db-url.sops.yaml in k8s/apps/home-assistant/app/"
Task: "Create Home Assistant Recorder configuration file configuration.yaml in k8s/apps/home-assistant/app/"
```

---

## Implementation Strategy

### MVP First (User Story 1 Only)

1. Complete Phase 1: Setup
2. Complete Phase 2: Foundational (CRITICAL - blocks all stories)
3. Complete Phase 3: User Story 1
4. **STOP and VALIDATE**: Test User Story 1 independently - verify history continuity
5. Deploy/demo if ready

### Incremental Delivery

1. Complete Setup + Foundational → Foundation ready
2. Add User Story 1 → Test independently → Deploy/Demo (MVP!)
3. Add User Story 2 → Test independently → Document rollback capability
4. Add User Story 3 → Test independently → Validate backup/restore
5. Each story adds value without breaking previous stories

### Sequential Execution (Recommended)

With a single operator:

1. Complete Setup + Foundational together
2. Execute User Story 1 (migration) → Verify success
3. Execute User Story 2 (rollback testing) → Verify rollback works
4. Execute User Story 3 (backup/restore validation) → Verify backups work
5. Complete Polish phase → Cleanup temporary resources

---

## Notes

- [P] tasks = different files, no dependencies
- [Story] label maps task to specific user story for traceability
- Each user story should be independently completable and testable
- Commit after each task or logical group
- Stop at any checkpoint to validate story independently
- Database content operations (creating DB/user) are operational steps; document actions taken
- Migration Job is temporary - remove from Git after successful cutover and 24h observation
- Avoid: vague tasks, same file conflicts, cross-story dependencies that break independence
