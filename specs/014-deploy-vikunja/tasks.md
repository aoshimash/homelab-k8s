# Tasks: Deploy Vikunja

**Input**: Design documents from `/specs/014-deploy-vikunja/`  
**Prerequisites**: plan.md, spec.md, research.md, data-model.md, contracts/, quickstart.md

**Organization**: Tasks are grouped by user story to enable independent implementation and testing of each story.

## Format: `[ID] [P?] [Story] Description`

- **[P]**: Can run in parallel (different files, no dependencies)
- **[Story]**: Which user story this task belongs to (e.g., US1, US2, US3)
- Include exact file paths in descriptions

## Phase 1: Setup (Shared Infrastructure)

**Purpose**: Project initialization and basic structure

- [ ] T001 Create namespace resource in `k8s/apps/vikunja/namespace.yaml`
- [ ] T002 [P] Create kustomization.yaml in `k8s/apps/vikunja/kustomization.yaml`
- [ ] T003 [P] Wire vikunja namespace into `k8s/apps/kustomization.yaml`

---

## Phase 2: Foundational (Blocking Prerequisites)

**Purpose**: Core infrastructure that MUST be complete before ANY user story can be implemented

**⚠️ CRITICAL**: No user story work can begin until this phase is complete

- [ ] T004 [P] Create SOPS-encrypted Secret for Vikunja DB role credentials in `k8s/configs/postgres/secret-vikunja-db.sops.yaml` (type: kubernetes.io/basic-auth, keys: username, password)
- [ ] T005 Update CloudNativePG Cluster spec to add managed role in `k8s/configs/postgres/cluster.yaml` (add spec.managed.roles[] entry for vikunja role with passwordSecret reference)
- [ ] T006 [P] Create CNPG Database CRD for Vikunja database in `k8s/configs/postgres/database-vikunja.yaml` (name: vikunja, owner: vikunja role)
- [ ] T007 [P] Create SOPS-encrypted Secret for Vikunja JWT secret in `k8s/apps/vikunja/app/secret-vikunja.sops.yaml`
- [ ] T008 [P] Create SOPS-encrypted Secret for Vikunja DB connection info in `k8s/apps/vikunja/app/secret-db.sops.yaml` (if not reusing postgres secret)

**Checkpoint**: Foundation ready - database and secrets are provisioned, user story implementation can now begin

---

## Phase 3: User Story 1 - Use Vikunja for personal task management (Priority: P1) 🎯 MVP

**Goal**: Deploy Vikunja service accessible via Tailscale Ingress, allowing users to sign in and manage tasks (projects, tasks, due dates, completion) to replace ad-hoc to-do notes.

**Independent Test**: Open the service via Tailscale, sign in with admin-created credentials, create a project and a task, sign out and back in, confirm the task remains available.

### Implementation for User Story 1

- [ ] T009 [P] [US1] Create Longhorn PVC for file storage in `k8s/apps/vikunja/app/pvc.yaml` (storageClass: longhorn, annotation: recurring-job-selector.longhorn.io for daily backups)
- [ ] T010 [P] [US1] Create HelmRepository resource for Vikunja chart in `k8s/apps/vikunja/app/helmrepository.yaml` (points to official Vikunja chart repository)
- [ ] T011 [US1] Create HelmRelease resource for Vikunja in `k8s/apps/vikunja/app/helmrelease.yaml` (configured for external PostgreSQL, existing PVC, registration disabled, Tailscale ingress)
- [ ] T012 [P] [US1] Create Tailscale Ingress resource in `k8s/apps/vikunja/app/ingress.yaml` (if chart ingress is not used, ingressClassName: tailscale, annotation: tailscale.com/proxy-group: ingress-proxies, host: vikunja)
- [ ] T013 [US1] Create app kustomization.yaml in `k8s/apps/vikunja/app/kustomization.yaml` (references all app resources)
- [ ] T014 [US1] Verify Flux reconciliation: check Kustomization and HelmRelease status are Ready
- [ ] T015 [US1] Verify pod readiness: check Vikunja pods are Running and Ready
- [ ] T016 [US1] Verify PVC binding: confirm vikunja-files PVC is Bound
- [ ] T017 [US1] Verify Ingress: confirm Ingress exists and host vikunja resolves within Tailscale
- [ ] T018 [US1] Smoke test: open https://vikunja/ from trusted network/VPN, verify login page loads
- [ ] T019 [US1] Create first admin user: execute `kubectl -n vikunja exec -it deploy/vikunja -- ./vikunja user create --email <email> --user <username> --password <password>`
- [ ] T020 [US1] End-to-end test: sign in, create a project, create a task, sign out, sign in again, verify task persists

**Checkpoint**: At this point, User Story 1 should be fully functional and testable independently - users can access Vikunja, sign in, and manage tasks

---

## Phase 4: User Story 2 - Collaborate on shared projects (Priority: P2)

**Goal**: Enable users to share projects with other users and collaborate on tasks, allowing multiple people to coordinate without duplicating effort.

**Independent Test**: Create two user accounts via CLI, User A shares a project with User B, User B can access and edit tasks in the shared project, User A sees updated task state.

### Implementation for User Story 2

- [ ] T021 [US2] Create second user account via CLI: execute `kubectl -n vikunja exec -it deploy/vikunja -- ./vikunja user create --email <email2> --user <username2> --password <password2>`
- [ ] T022 [US2] Verify sharing functionality: User A shares a project with User B via UI
- [ ] T023 [US2] Verify collaborative editing: User B edits/completes a task, User A sees updated state and content
- [ ] T024 [US2] Verify edit permissions: confirm shared users can create, edit, and complete tasks (per FR-013)

**Checkpoint**: At this point, User Stories 1 AND 2 should both work independently - users can manage personal tasks and collaborate on shared projects

---

## Phase 5: User Story 3 - Operate the service safely (Priority: P3)

**Goal**: Ensure the Vikunja service can be maintained (restarted, upgraded, recovered) without losing task data, making the system trustworthy over time.

**Independent Test**: Create sample tasks/projects, perform controlled restart/upgrade, verify data persists, perform restore drill from backups, verify restored data contains expected tasks/projects.

### Implementation for User Story 3

- [ ] T025 [US3] Verify data persistence across restart: create sample tasks/projects, restart Vikunja pods, verify data remains intact
- [ ] T026 [US3] Verify CNPG scheduled backups: confirm ScheduledBackup runs successfully daily (check CNPG status/events/logs)
- [ ] T027 [US3] Verify Longhorn volume backups: confirm daily backups exist for vikunja-files PVC (check Longhorn UI/API)
- [ ] T028 [US3] Perform database restore drill: restore Vikunja DB from CNPG backup, verify restored DB contains expected sample data, verify application can connect and sign-in works
- [ ] T029 [US3] Perform files restore drill: restore vikunja-files volume from Longhorn backup, verify restored volume is mountable, verify sample attachment file is present and accessible
- [ ] T030 [US3] Document restore procedure: create runbook documenting restore steps per `contracts/backup-restore.md`

**Checkpoint**: All user stories should now be independently functional - service is deployable, collaborative, and recoverable

---

## Phase 6: Polish & Cross-Cutting Concerns

**Purpose**: Improvements that affect multiple user stories

- [ ] T031 [P] Verify all health signals per `contracts/reconciliation.md`: Flux Ready, pods Running, PVC Bound, Ingress reachable, login page loads
- [ ] T032 [P] Run quickstart.md validation: execute all quickstart steps and verify they work as documented
- [ ] T033 [P] Verify backup success signals: CNPG ScheduledBackup runs daily, Longhorn daily backups exist for files volume
- [ ] T034 [P] Security review: confirm no public exposure (Tailscale only), registration disabled, secrets encrypted with SOPS
- [ ] T035 [P] Documentation updates: ensure all operational procedures are documented and match actual configuration

---

## Dependencies & Execution Order

### Phase Dependencies

- **Setup (Phase 1)**: No dependencies - can start immediately
- **Foundational (Phase 2)**: Depends on Setup completion - BLOCKS all user stories
- **User Stories (Phase 3+)**: All depend on Foundational phase completion
  - User stories can then proceed sequentially in priority order (P1 → P2 → P3)
- **Polish (Final Phase)**: Depends on all desired user stories being complete

### User Story Dependencies

- **User Story 1 (P1)**: Can start after Foundational (Phase 2) - No dependencies on other stories
- **User Story 2 (P2)**: Can start after User Story 1 - Requires US1 to be functional (needs user accounts and projects)
- **User Story 3 (P3)**: Can start after User Story 1 - Requires US1 to be functional (needs sample data for backup/restore testing)

### Within Each User Story

- Infrastructure resources (PVC, HelmRepository) before application resources (HelmRelease)
- Application deployment before verification tasks
- Verification before end-to-end testing
- Story complete before moving to next priority

### Parallel Opportunities

- All Setup tasks marked [P] can run in parallel (T002, T003)
- All Foundational tasks marked [P] can run in parallel within Phase 2 (T004, T006, T007, T008)
- Within User Story 1, tasks marked [P] can run in parallel (T009, T010, T012)
- Polish phase tasks marked [P] can run in parallel (T031-T035)

---

## Parallel Example: User Story 1

```bash
# Launch foundational resources in parallel:
Task: "Create SOPS-encrypted Secret for Vikunja DB role credentials in k8s/configs/postgres/secret-vikunja-db.sops.yaml"
Task: "Create CNPG Database CRD for Vikunja database in k8s/configs/postgres/database-vikunja.yaml"
Task: "Create SOPS-encrypted Secret for Vikunja JWT secret in k8s/apps/vikunja/app/secret-vikunja.sops.yaml"
Task: "Create SOPS-encrypted Secret for Vikunja DB connection info in k8s/apps/vikunja/app/secret-db.sops.yaml"

# Launch US1 infrastructure in parallel:
Task: "Create Longhorn PVC for file storage in k8s/apps/vikunja/app/pvc.yaml"
Task: "Create HelmRepository resource for Vikunja chart in k8s/apps/vikunja/app/helmrepository.yaml"
Task: "Create Tailscale Ingress resource in k8s/apps/vikunja/app/ingress.yaml"
```

---

## Implementation Strategy

### MVP First (User Story 1 Only)

1. Complete Phase 1: Setup
2. Complete Phase 2: Foundational (CRITICAL - blocks all stories)
3. Complete Phase 3: User Story 1
4. **STOP and VALIDATE**: Test User Story 1 independently (sign in, create project/task, verify persistence)
5. Deploy/demo if ready

### Incremental Delivery

1. Complete Setup + Foundational → Foundation ready (database, secrets provisioned)
2. Add User Story 1 → Test independently → Deploy/Demo (MVP!)
3. Add User Story 2 → Test independently → Deploy/Demo (collaboration enabled)
4. Add User Story 3 → Test independently → Deploy/Demo (backup/restore validated)
5. Each story adds value without breaking previous stories

### Sequential Strategy (Recommended for GitOps)

With GitOps workflow:
1. Complete Setup + Foundational together
2. Commit and let Flux reconcile
3. Once Foundational is verified:
   - Implement User Story 1
   - Commit and reconcile
   - Test independently
4. Then User Story 2 (depends on US1 for user accounts)
5. Then User Story 3 (depends on US1 for sample data)

---

## Notes

- [P] tasks = different files, no dependencies
- [Story] label maps task to specific user story for traceability
- Each user story should be independently completable and testable
- Commit after each phase or logical group
- Stop at any checkpoint to validate story independently
- Verify Flux reconciliation after each commit
- Avoid: vague tasks, same file conflicts, cross-story dependencies that break independence
- All secrets MUST be SOPS-encrypted before committing
- All changes MUST be declarative (no imperative kubectl commands for persistent state)
