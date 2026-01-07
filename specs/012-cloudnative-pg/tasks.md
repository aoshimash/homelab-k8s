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

- [ ] T001 Create directory structure `k8s/infrastructure/cloudnative-pg/`
- [ ] T002 Create directory structure `k8s/configs/postgres/`
- [ ] T003 [P] Create namespace manifest in `k8s/infrastructure/cloudnative-pg/namespace.yaml`
- [ ] T004 [P] Create root kustomization in `k8s/infrastructure/cloudnative-pg/kustomization.yaml`
- [ ] T005 Update infrastructure kustomization to include cloudnative-pg in `k8s/infrastructure/kustomization.yaml`
- [ ] T006 [P] Create namespace manifest in `k8s/configs/postgres/namespace.yaml`
- [ ] T007 [P] Create root kustomization in `k8s/configs/postgres/kustomization.yaml`
- [ ] T008 Update configs kustomization to include postgres in `k8s/configs/kustomization.yaml`

**Checkpoint**: Directory structure ready for resource manifests

---

## Phase 2: Foundational (CloudNativePG Operator Installation)

**Purpose**: Deploy CloudNativePG operator that MUST be available before Cluster CRD can be created

**⚠️ CRITICAL**: Operator must be running and CRDs installed before PostgreSQL cluster can be deployed

- [ ] T009 [P] Create HelmRepository manifest in `k8s/infrastructure/cloudnative-pg/helmrepository.yaml`
- [ ] T010 [P] Create HelmRelease manifest for operator in `k8s/infrastructure/cloudnative-pg/helmrelease.yaml`
- [ ] T011 Verify operator deployment with `kubectl get deploy -n cnpg-system cnpg-controller-manager`
- [ ] T012 Verify CRDs installed with `kubectl get crd clusters.postgresql.cnpg.io`

**Checkpoint**: Operator foundation ready - PostgreSQL cluster implementation can begin

---

## Phase 3: User Story 1 - Shared PostgreSQL Database Cluster (Priority: P1) 🎯 MVP

**Goal**: Deploy PostgreSQL cluster with persistent storage and verify it reaches healthy state

**Independent Test**: Deploy the PostgreSQL cluster and verify it is running and accepting connections from within the cluster using the default superuser credentials.

**Prerequisites**: Phase 2 complete (CloudNativePG operator running)

### Implementation for User Story 1

- [ ] T013 [P] [US1] Create Cluster CRD manifest in `k8s/configs/postgres/cluster.yaml`
- [ ] T014 [US1] Update postgres kustomization to include cluster.yaml in `k8s/configs/postgres/kustomization.yaml`
- [ ] T015 [US1] Verify cluster status with `kubectl get cluster -n postgres postgres-cluster`
- [ ] T016 [US1] Verify pod running with `kubectl get pods -n postgres -l cnpg.io/cluster=postgres-cluster`
- [ ] T017 [US1] Verify PVC bound with `kubectl get pvc -n postgres`
- [ ] T018 [US1] Verify services created with `kubectl get svc -n postgres` (should show rw, r, ro services)
- [ ] T019 [US1] Get superuser credentials with `kubectl get secret -n postgres postgres-cluster-superuser -o jsonpath='{.data.password}' | base64 -d`
- [ ] T020 [US1] Test connection from within cluster using psql pod
- [ ] T021 [US1] Verify cluster health status shows "Cluster in healthy state"

**Checkpoint**: User Story 1 complete - PostgreSQL cluster running, accepting connections, data persisted on Longhorn

---

## Phase 4: User Story 4 - Automated Backups (Priority: P4)

**Goal**: Configure automated daily backups to R2 with 7-day retention

**Independent Test**: Configure backup settings and verify that backups are created successfully to the Longhorn R2 backup target.

**Prerequisites**: User Story 1 complete (PostgreSQL cluster running)

### Implementation for User Story 4

- [ ] T022 [P] [US4] Create R2 credentials secret (SOPS encrypted) in `k8s/configs/postgres/secret-r2-credentials.sops.yaml`
- [ ] T023 [P] [US4] Create ScheduledBackup CRD manifest in `k8s/configs/postgres/scheduledbackup.yaml`
- [ ] T024 [US4] Update postgres kustomization to include secret and scheduledbackup in `k8s/configs/postgres/kustomization.yaml`
- [ ] T025 [US4] Verify ScheduledBackup created with `kubectl get scheduledbackup -n postgres`
- [ ] T026 [US4] Wait for first scheduled backup or trigger manual backup
- [ ] T027 [US4] Verify backup completed with `kubectl get backup -n postgres`
- [ ] T028 [US4] Verify backup stored in R2 (check backup details)

**Checkpoint**: User Story 4 complete - Automated backups configured and verified

---

## Phase 5: Documentation

**Purpose**: Create operational documentation for cluster management

- [ ] T029 Create operational documentation in `docs/cloudnative-pg.md`
- [ ] T030 Document connection instructions for applications
- [ ] T031 Document database/user provisioning procedure (for User Story 2)
- [ ] T032 Document backup/restore procedures
- [ ] T033 Document troubleshooting guide

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
