# Tasks: kube-state-metrics

**Input**: Design documents from `/specs/006-kube-metrics/`  
**Prerequisites**: plan.md (required), spec.md (required for user stories), research.md, data-model.md, contracts/

**Tests**: Not required - manual verification via kubectl and Grafana Cloud queries.

**Organization**: Tasks are grouped by user story to enable independent implementation and testing of each story.

## Format: `[ID] [P?] [Story] Description`

- **[P]**: Can run in parallel (different files, no dependencies)
- **[Story]**: Which user story this task belongs to (e.g., US1, US2, US3)
- Include exact file paths in descriptions

## Path Conventions

- **Infrastructure manifests**: `k8s/infrastructure/`
- **Documentation**: `docs/`
- **Feature specs**: `specs/006-kube-metrics/`

---

## Phase 1: Setup (kube-state-metrics Directory Structure)

**Purpose**: Create directory structure and foundational manifests for kube-state-metrics

- [ ] T001 Create directory `k8s/infrastructure/kube-state-metrics/`
- [ ] T002 [P] Create HelmRepository for prometheus-community in `k8s/infrastructure/kube-state-metrics/helmrepository.yaml`
- [ ] T003 [P] Create Kustomization file in `k8s/infrastructure/kube-state-metrics/kustomization.yaml`

---

## Phase 2: User Story 2 - GitOps-managed kube-state-metrics Deployment (Priority: P1) 🎯 MVP

**Goal**: Deploy kube-state-metrics via GitOps (Flux) following the same patterns as other infrastructure components.

**Independent Test**: After committing manifests to Git, Flux reconciles and kube-state-metrics runs in `monitoring` namespace without manual intervention.

### Implementation for User Story 2

- [ ] T004 [US2] Create HelmRelease for kube-state-metrics in `k8s/infrastructure/kube-state-metrics/helmrelease.yaml`
- [ ] T005 [US2] Update infrastructure Kustomization to include kube-state-metrics in `k8s/infrastructure/kustomization.yaml`
- [ ] T006 [US2] Commit and push changes to trigger Flux reconciliation
- [ ] T007 [US2] Verify HelmRepository is Ready via `kubectl get helmrepository prometheus-community -n flux-system`
- [ ] T008 [US2] Verify HelmRelease is Ready via `kubectl get helmrelease kube-state-metrics -n monitoring`
- [ ] T009 [US2] Verify kube-state-metrics pod is Running via `kubectl get pods -n monitoring -l app.kubernetes.io/name=kube-state-metrics`

**Checkpoint**: kube-state-metrics is deployed via GitOps. User Story 2 complete.

---

## Phase 3: User Story 3 - Integrate kube-state-metrics with Grafana Alloy (Priority: P1)

**Goal**: Configure Grafana Alloy to scrape kube-state-metrics and send metrics to Grafana Cloud with `cluster=homelab` label.

**Independent Test**: kube-state-metrics are queryable in Grafana Cloud Prometheus with the `cluster=homelab` label.

### Implementation for User Story 3

- [ ] T010 [US3] Add kube-state-metrics service discovery config to Grafana Alloy in `k8s/infrastructure/grafana-alloy/helmrelease.yaml`
- [ ] T011 [US3] Add kube-state-metrics scrape config to Grafana Alloy in `k8s/infrastructure/grafana-alloy/helmrelease.yaml`
- [ ] T012 [US3] Commit and push Grafana Alloy config changes
- [ ] T013 [US3] Verify Grafana Alloy pods restart and are Ready via `kubectl get pods -n monitoring -l app.kubernetes.io/name=alloy`
- [ ] T014 [US3] Verify Alloy is scraping kube-state-metrics via `kubectl logs -n monitoring -l app.kubernetes.io/name=alloy | grep kube-state`

**Checkpoint**: Grafana Alloy is scraping kube-state-metrics. User Story 3 complete.

---

## Phase 4: User Story 1 - Monitor Kubernetes Resource States (Priority: P1)

**Goal**: Verify kube_* metrics are visible in Grafana Cloud with accurate values reflecting cluster state.

**Independent Test**: Grafana Cloud displays metrics such as `kube_deployment_spec_replicas`, `kube_pod_status_phase`, and `kube_node_status_condition` with accurate values.

### Verification for User Story 1

- [ ] T015 [US1] Query `kube_deployment_spec_replicas{cluster="homelab"}` in Grafana Cloud and verify results
- [ ] T016 [US1] Query `kube_pod_status_phase{cluster="homelab"}` in Grafana Cloud and verify pod states match `kubectl get pods -A`
- [ ] T017 [US1] Query `kube_node_status_condition{cluster="homelab"}` in Grafana Cloud and verify node conditions
- [ ] T018 [US1] Test metric accuracy by scaling a deployment and verifying metrics update within 5 minutes

**Checkpoint**: All kube_* metrics visible in Grafana Cloud with correct labels. User Story 1 complete.

---

## Phase 5: Polish & Documentation

**Purpose**: Add operational documentation and finalize implementation

- [ ] T019 [P] Create operational documentation in `docs/kube-state-metrics.md`
- [ ] T020 [P] Update tasks.md to mark all tasks as complete in `specs/006-kube-metrics/tasks.md`
- [ ] T021 Run quickstart.md validation steps to ensure documentation accuracy

---

## Dependencies & Execution Order

### Phase Dependencies

- **Setup (Phase 1)**: No dependencies - can start immediately
- **User Story 2 (Phase 2)**: Depends on Setup (Phase 1) completion - This is the MVP
- **User Story 3 (Phase 3)**: Depends on User Story 2 (kube-state-metrics must be deployed before Alloy can scrape it)
- **User Story 1 (Phase 4)**: Depends on User Story 3 (metrics must flow to Grafana Cloud before verification)
- **Polish (Phase 5)**: Depends on all user stories being complete

### User Story Dependencies

```
Phase 1: Setup
     │
     ▼
Phase 2: User Story 2 (Deploy kube-state-metrics) ← MVP
     │
     ▼
Phase 3: User Story 3 (Integrate with Grafana Alloy)
     │
     ▼
Phase 4: User Story 1 (Verify metrics in Grafana Cloud)
     │
     ▼
Phase 5: Polish & Documentation
```

**Note**: Unlike typical applications, these user stories have sequential dependencies because:
- US2 (deployment) must complete before US3 (integration) can begin
- US3 (integration) must complete before US1 (verification) can begin

### Parallel Opportunities

- **Phase 1**: T002 and T003 can run in parallel (different files)
- **Phase 5**: T019 and T020 can run in parallel (different files)

---

## Implementation Strategy

### MVP First (User Story 2 Only)

1. Complete Phase 1: Setup
2. Complete Phase 2: User Story 2 (Deploy kube-state-metrics)
3. **STOP and VALIDATE**: Verify kube-state-metrics pod is running
4. This provides a functional kube-state-metrics deployment even without Alloy integration

### Full Implementation

1. Complete Setup → Directory structure ready
2. Complete User Story 2 → kube-state-metrics deployed via GitOps
3. Complete User Story 3 → Metrics flowing to Grafana Cloud
4. Complete User Story 1 → Metrics verified in Grafana Cloud
5. Complete Polish → Documentation finalized

### Estimated Time

| Phase | Tasks | Estimated Time |
|-------|-------|----------------|
| Phase 1: Setup | 3 tasks | 10 minutes |
| Phase 2: US2 | 6 tasks | 20 minutes (includes Flux reconciliation wait) |
| Phase 3: US3 | 5 tasks | 15 minutes (includes Alloy restart wait) |
| Phase 4: US1 | 4 tasks | 15 minutes (includes metric propagation wait) |
| Phase 5: Polish | 3 tasks | 15 minutes |
| **Total** | **21 tasks** | **~75 minutes** |

---

## Notes

- [P] tasks = different files, no dependencies
- [Story] label maps task to specific user story for traceability
- Verification tasks (T007-T009, T013-T014, T015-T018) require cluster access
- Grafana Cloud queries require Grafana Cloud access
- Allow 5-10 minutes for Flux reconciliation and metric propagation
- Commit after each logical group of tasks for GitOps workflow
