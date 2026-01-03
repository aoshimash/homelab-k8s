# Tasks: Grafana Alloy for Metrics and Logs

**Input**: Design documents from `/specs/004-grafana-alloy/`
**Prerequisites**: plan.md (required), spec.md (required for user stories), research.md, data-model.md, contracts/

**Tests**: Manual verification via Grafana Cloud dashboards, kubectl status checks, log queries in Loki (per plan.md)

**Organization**: Tasks are grouped by user story to enable independent implementation and testing of each story.

## Format: `[ID] [P?] [Story] Description`

- **[P]**: Can run in parallel (different files, no dependencies)
- **[Story]**: Which user story this task belongs to (e.g., US1, US2, US3)
- Include exact file paths in descriptions

## Path Conventions

Based on plan.md Project Structure:

- **Infrastructure manifests**: `k8s/infrastructure/grafana-alloy/`
- **Documentation**: `docs/grafana-alloy.md`
- **Flux config**: `k8s/flux/` (existing, update if needed)

---

## Phase 1: Setup (Project Structure)

**Purpose**: Create directory structure and base files for Grafana Alloy deployment

- [x] T001 Create `k8s/infrastructure/grafana-alloy/` directory
- [x] T002 [P] Create namespace manifest in `k8s/infrastructure/grafana-alloy/namespace.yaml`
- [x] T003 [P] Create HelmRepository manifest in `k8s/infrastructure/grafana-alloy/helmrepository.yaml`
- [x] T004 Create Kustomization entry point in `k8s/infrastructure/grafana-alloy/kustomization.yaml`

---

## Phase 2: Foundational (Blocking Prerequisites)

**Purpose**: Infrastructure that MUST be complete before Grafana Cloud integration

**⚠️ CRITICAL**: No user story work can begin until credentials are configured

- [x] T005 Obtain Grafana Cloud credentials (Access Policy Token with metrics:write, logs:write scopes) - manual step per quickstart.md
- [x] T006 Create unencrypted secret YAML with Grafana Cloud credentials (temporary, local only)
- [x] T007 Encrypt secret with SOPS and save to `k8s/infrastructure/grafana-alloy/secret-grafana-cloud.sops.yaml`
- [x] T008 Delete unencrypted secret file (security cleanup)
- [x] T009 Update `k8s/infrastructure/kustomization.yaml` to include `grafana-alloy/` directory

**Checkpoint**: Foundation ready - Grafana Alloy manifests can now be created

---

## Phase 3: User Story 1 & 2 - Send Metrics and Logs to Grafana Cloud (Priority: P1) 🎯 MVP

**Goal**: Deploy Grafana Alloy to collect and send Kubernetes metrics (US1) and logs (US2) to Grafana Cloud

**Note**: US1 (metrics) and US2 (logs) are both P1 and tightly coupled in Alloy configuration, so they are implemented together as MVP

**Independent Test**: 
- Query `up{cluster="homelab"}` in Grafana Cloud Prometheus shows cluster nodes
- Query `{cluster="homelab"}` in Grafana Cloud Loki shows cluster logs

### Implementation for User Story 1 & 2

- [x] T010 [US1/US2] Create HelmRelease manifest with DaemonSet controller type in `k8s/infrastructure/grafana-alloy/helmrelease.yaml`
- [x] T011 [US1] Add metrics collection River config (discovery.kubernetes nodes, prometheus.scrape kubelet) in HelmRelease values
- [x] T012 [US1] Add metrics remote_write config (prometheus.remote_write grafana_cloud) with Basic Auth in HelmRelease values
- [x] T013 [US1] Add cadvisor metrics scraping config (prometheus.scrape cadvisor) in HelmRelease values
- [x] T014 [US1] Add cluster label (cluster=homelab) via prometheus.relabel in HelmRelease values
- [x] T015 [US2] Add log collection River config (loki.source.kubernetes pods) in HelmRelease values
- [x] T016 [US2] Add log processing config (loki.process add_labels with cluster=homelab) in HelmRelease values
- [x] T017 [US2] Add log write config (loki.write grafana_cloud) with Basic Auth in HelmRelease values
- [x] T018 [US1/US2] Configure environment variable injection from Secret for credentials in HelmRelease

**Checkpoint**: At this point, Metrics (US1) and Logs (US2) should be fully configured in HelmRelease

---

## Phase 4: User Story 3 - GitOps-managed Deployment (Priority: P2)

**Goal**: Deploy Grafana Alloy via Flux reconciliation with encrypted credentials

**Independent Test**: 
- `flux get kustomizations infrastructure` shows Ready
- `flux get helmreleases -n monitoring` shows Ready
- `kubectl get daemonset -n monitoring` shows Alloy pods running

### Implementation for User Story 3

- [x] T019 [US3] Verify Kustomization includes all resources (namespace, helmrepository, helmrelease, secret) in `k8s/infrastructure/grafana-alloy/kustomization.yaml`
- [ ] T020 [US3] Commit all manifests to Git repository
- [ ] T021 [US3] Push changes to trigger Flux reconciliation
- [ ] T022 [US3] Verify Flux reconciliation status with `flux get kustomizations infrastructure`
- [ ] T023 [US3] Verify HelmRelease status with `flux get helmreleases -n monitoring`
- [ ] T024 [US3] Verify DaemonSet is running with `kubectl get daemonset -n monitoring`
- [ ] T025 [US3] Check Alloy pod logs for successful startup and data transmission

**Checkpoint**: Grafana Alloy is deployed and running via GitOps

---

## Phase 5: Verification & Validation

**Purpose**: End-to-end validation of metrics and logs in Grafana Cloud

- [ ] T026 [US1] Verify node metrics in Grafana Cloud: query `up{cluster="homelab"}`
- [ ] T027 [US1] Verify container metrics in Grafana Cloud: query `container_cpu_usage_seconds_total{cluster="homelab"}`
- [ ] T028 [US2] Verify logs in Grafana Cloud Loki: query `{cluster="homelab"}`
- [ ] T029 [US2] Verify log labels (namespace, pod, container) are present and filterable
- [ ] T030 Verify no plaintext credentials in Git repository (grep for API key patterns)

---

## Phase 6: Documentation & Polish

**Purpose**: Complete operational documentation

- [x] T031 [P] Create operational documentation in `docs/grafana-alloy.md` based on quickstart.md
- [x] T032 [P] Document Grafana Cloud Access Policy Token creation steps in `docs/grafana-alloy.md`
- [x] T033 [P] Document credential rotation procedure in `docs/grafana-alloy.md`
- [x] T034 [P] Document troubleshooting steps (no metrics, no logs, auth errors) in `docs/grafana-alloy.md`
- [ ] T035 Update plan.md to mark Phase 2 tasks complete

---

## Dependencies & Execution Order

### Phase Dependencies

- **Setup (Phase 1)**: No dependencies - can start immediately
- **Foundational (Phase 2)**: Depends on Phase 1 + Grafana Cloud account access
- **US1/US2 (Phase 3)**: Depends on Phase 2 (credentials must exist)
- **US3 (Phase 4)**: Depends on Phase 3 (manifests must be ready)
- **Verification (Phase 5)**: Depends on Phase 4 (deployment must be running)
- **Documentation (Phase 6)**: Can start after Phase 3, finalize after Phase 5

### User Story Dependencies

- **User Story 1 (Metrics) + User Story 2 (Logs)**: Both P1, implemented together in Phase 3
  - Tightly coupled in Alloy configuration
  - Single HelmRelease handles both
  - Can be tested independently in Grafana Cloud
- **User Story 3 (GitOps)**: P2, depends on US1/US2 manifests being ready
  - Tests deployment workflow, not functionality

### Parallel Opportunities

```text
Phase 1 (all can run in parallel):
  T002 || T003  (namespace.yaml || helmrepository.yaml)

Phase 3 (within HelmRelease, conceptually parallel sections):
  T011-T014 (metrics config) can be written as a unit
  T015-T017 (logs config) can be written as a unit
  Then T018 (env injection) connects them

Phase 6 (all documentation tasks):
  T031 || T032 || T033 || T034  (different doc sections)
```

---

## Implementation Strategy

### MVP First (Phase 1-4)

1. Complete Phase 1: Setup (directory and base files)
2. Complete Phase 2: Foundational (credentials encrypted)
3. Complete Phase 3: US1/US2 (HelmRelease with metrics + logs)
4. Complete Phase 4: US3 (GitOps deployment)
5. **STOP and VALIDATE**: Verify data in Grafana Cloud
6. Proceed to Phase 5 & 6 for validation and documentation

### Single Developer Flow

```text
Day 1:
  - T001-T004: Setup directory structure
  - T005-T009: Obtain and encrypt credentials
  - T010-T018: Create HelmRelease with full config

Day 2:
  - T019-T025: Commit, push, verify deployment
  - T026-T030: Validate in Grafana Cloud
  - T031-T035: Documentation
```

### Estimated Effort

- **Phase 1 (Setup)**: 15 minutes
- **Phase 2 (Foundational)**: 30 minutes (includes Grafana Cloud console work)
- **Phase 3 (US1/US2)**: 1-2 hours (River config complexity)
- **Phase 4 (US3)**: 30 minutes
- **Phase 5 (Verification)**: 30 minutes
- **Phase 6 (Documentation)**: 1 hour

**Total**: ~4-5 hours

---

## Notes

- [P] tasks = different files, no dependencies
- [Story] label maps task to specific user story for traceability
- US1 and US2 are implemented together due to shared Alloy configuration
- River configuration syntax should be validated before commit
- All credentials must be encrypted before committing to Git
- Verify SOPS decryption is working in Flux before expecting deployment to succeed
- Refer to `quickstart.md` for detailed step-by-step instructions
- Refer to `contracts/reconciliation.md` for expected Flux behavior
