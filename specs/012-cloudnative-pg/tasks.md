# Tasks: CloudNativePG PostgreSQL Cluster

**Input**: Design documents from `/specs/012-cloudnative-pg/`
**Prerequisites**: plan.md, spec.md, research.md, data-model.md, contracts/

**Tests**: Manual verification via kubectl (no automated tests required)

**Organization**: Tasks are grouped by user story to enable independent implementation and testing of each story.

## Format: `[ID] [P?] [Story] Description`

- **[P]**: Can run in parallel (different files, no dependencies)
- **[Story]**: Which user story this task belongs to (e.g., US1, US4)
- Include exact file paths in descriptions

## Path Conventions

- **Infrastructure manifests**: `k8s/infrastructure/cloudnative-pg/`
- **Config manifests**: `k8s/configs/postgres/`
- **Documentation**: `docs/`
- Follows existing `longhorn` and `tailscale` patterns in the codebase

---

## Phase 1: Setup (Directory Structure)

**Purpose**: Create directory structure and base kustomization files

- [x] T001 Create directory structure `k8s/infrastructure/cloudnative-pg/`
- [x] T002 Create directory structure `k8s/configs/postgres/`
- [x] T003 [P] Create namespace manifest in `k8s/infrastructure/cloudnative-pg/namespace.yaml`
- [x] T004 [P] Create root kustomization in `k8s/infrastructure/cloudnative-pg/kustomization.yaml`
- [x] T005 Update infrastructure kustomization to include cloudnative-pg in `k8s/infrastructure/kustomization.yaml`
- [x] T006 [P] Create namespace manifest in `k8s/configs/postgres/namespace.yaml`
- [x] T007 [P] Create root kustomization in `k8s/configs/postgres/kustomization.yaml`
- [x] T008 Update configs kustomization to include postgres in `k8s/configs/kustomization.yaml`

**Checkpoint**: Directory structure ready for resource manifests

---

## Phase 2: Foundational (CloudNativePG Operator Installation)

**Purpose**: Deploy CloudNativePG operator that MUST be available before Cluster CRD can be created

**⚠️ CRITICAL**: Operator must be running and CRDs installed before PostgreSQL cluster can be deployed

- [x] T009 [P] Create HelmRepository manifest in `k8s/infrastructure/cloudnative-pg/helmrepository.yaml`
- [x] T010 [P] Create HelmRelease manifest for operator in `k8s/infrastructure/cloudnative-pg/helmrelease.yaml`
- [x] T011 Verify operator deployment with `kubectl get deploy -n cnpg-system cloudnative-pg` (deployment name is `cloudnative-pg`, not `cnpg-controller-manager`)
- [x] T012 Verify CRDs installed with `kubectl get crd clusters.postgresql.cnpg.io`

**Checkpoint**: Operator foundation ready - PostgreSQL cluster implementation can begin

---

## Phase 3: User Story 1 - Shared PostgreSQL Database Cluster (Priority: P1) 🎯 MVP

**Goal**: Deploy PostgreSQL cluster with persistent storage and verify it reaches healthy state

**Independent Test**: Deploy the PostgreSQL cluster and verify it is running and accepting connections from within the cluster using the default superuser credentials.

**Prerequisites**: Phase 2 complete (CloudNativePG operator running)

### Implementation for User Story 1

- [x] T013 [P] [US1] Create Cluster CRD manifest in `k8s/configs/postgres/cluster.yaml`
- [x] T014 [US1] Update postgres kustomization to include cluster.yaml in `k8s/configs/postgres/kustomization.yaml`
- [x] T015 [US1] Verify cluster status with `kubectl get cluster -n postgres postgres-cluster` (STATUS: Cluster in healthy state)
- [x] T016 [US1] Verify pod running with `kubectl get pods -n postgres -l cnpg.io/cluster=postgres-cluster` (postgres-cluster-1 Running)
- [x] T017 [US1] Verify PVC bound with `kubectl get pvc -n postgres` (postgres-cluster-1 Bound, 10Gi)
- [x] T018 [US1] Verify services created with `kubectl get svc -n postgres` (rw, r, ro services created)
- [x] T019 [US1] Get app user credentials with `kubectl get secret -n postgres postgres-cluster-app` (superuser access disabled by default in newer versions)
- [x] T020 [US1] Test connection from within cluster using psql pod (PostgreSQL 16.11 connection successful)
- [x] T021 [US1] Verify cluster health status shows "Cluster in healthy state" (Ready: True)

**Checkpoint**: User Story 1 complete - PostgreSQL cluster running, accepting connections, data persisted on Longhorn

---

## Phase 4: User Story 4 - Automated Backups (Priority: P4)

**Goal**: Configure automated daily backups to R2 with 7-day retention

**Independent Test**: Configure backup settings and verify that backups are created successfully to the Longhorn R2 backup target.

**Prerequisites**: User Story 1 complete (PostgreSQL cluster running)

### Implementation for User Story 4

- [x] T022 [P] [US4] Create R2 credentials secret (SOPS encrypted) in `k8s/configs/postgres/secret-r2-credentials.sops.yaml`
- [x] T023 [P] [US4] Create ScheduledBackup CRD manifest in `k8s/configs/postgres/scheduledbackup.yaml`
- [x] T024 [US4] Update postgres kustomization to include secret and scheduledbackup in `k8s/configs/postgres/kustomization.yaml`
- [x] T025 [US4] Verify ScheduledBackup created with `kubectl get scheduledbackup -n postgres` (postgres-cluster-daily created)
- [ ] T026 [US4] Wait for first scheduled backup or trigger manual backup (scheduled for 00:00 UTC daily)
- [ ] T027 [US4] Verify backup completed with `kubectl get backup -n postgres` (waiting for first backup)
- [ ] T028 [US4] Verify backup stored in R2 (check backup details after first backup)

**Checkpoint**: User Story 4 complete - Automated backups configured and verified

---

## Phase 5: Documentation

**Purpose**: Create operational documentation for cluster management

- [x] T029 Create operational documentation in `docs/cloudnative-pg.md`
- [x] T030 Document connection instructions for applications
- [x] T031 Document database/user provisioning procedure (for User Story 2)
- [x] T032 Document backup/restore procedures
- [x] T033 Document troubleshooting guide

**Checkpoint**: Documentation complete - operators can manage cluster independently

---

## Dependency Graph

```
Phase 1 (Setup)
    │
    ▼
Phase 2 (Operator) ──┐
    │                │
    ▼                │
Phase 3 (US1) ───────┘
    │
    ▼
Phase 4 (US4)
    │
    ▼
Phase 5 (Documentation)
```

**Story Completion Order**:
1. **US1** (P1) - Must complete first (core functionality)
2. **US4** (P4) - Depends on US1 (backups require running cluster)
3. **US2** (P2) - Manual SQL operations (documented, not implemented)
4. **US3** (P3) - Included in US1 (storage configured in Cluster CRD)

## Parallel Execution Opportunities

### Phase 1
- T003, T004, T006, T007 can run in parallel (different directories)

### Phase 2
- T009, T010 can run in parallel (different resources)

### Phase 3
- T013 can run independently (Cluster CRD creation)

### Phase 4
- T022, T023 can run in parallel (different resources)

## Implementation Strategy

### MVP Scope
**Minimum Viable Product**: Phase 1 + Phase 2 + Phase 3 (User Story 1)
- Operator installed
- PostgreSQL cluster running
- Persistent storage configured
- Cluster accepting connections

This delivers a functional PostgreSQL cluster that applications can use.

### Incremental Delivery
1. **Increment 1**: MVP (US1) - Core database cluster
2. **Increment 2**: Backups (US4) - Production readiness
3. **Increment 3**: Documentation - Operational maturity

### Notes
- **User Story 2** (Application Database Provisioning): Manual SQL operations documented in quickstart.md, not automated tasks
- **User Story 3** (Persistent Storage): Handled automatically by Cluster CRD with Longhorn storageClass
- All tasks follow GitOps-first principle: manifests committed to Git, Flux reconciles automatically
