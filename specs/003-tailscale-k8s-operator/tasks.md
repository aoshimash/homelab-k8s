# Tasks: Tailscale Kubernetes Operator

**Input**: Design documents from `/specs/003-tailscale-k8s-operator/`
**Prerequisites**: plan.md, spec.md, research.md, data-model.md, contracts/reconciliation.md

**Tests**: Manual verification via kubectl and tailnet access (no automated tests)

**Organization**: Tasks are grouped by user story to enable independent implementation and testing of each story.

## Format: `[ID] [P?] [Story] Description`

- **[P]**: Can run in parallel (different files, no dependencies)
- **[Story]**: Which user story this task belongs to (e.g., US1, US2, US3, US4)
- Include exact file paths in descriptions

## Path Conventions

- **Infrastructure manifests**: `k8s/infrastructure/tailscale/`
- **Smoke test resources**: `k8s/infrastructure/tailscale/smoke/`
- **Cilium config**: `k8s/infrastructure/cilium/`

---

## Phase 1: Setup (Shared Infrastructure)

**Purpose**: Create directory structure and base manifests for Tailscale Operator

- [x] T001 Create tailscale directory structure in `k8s/infrastructure/tailscale/`
- [x] T002 [P] Create namespace manifest in `k8s/infrastructure/tailscale/namespace.yaml`
- [x] T003 [P] Create HelmRepository for Tailscale charts in `k8s/infrastructure/tailscale/helmrepository.yaml`

---

## Phase 2: Foundational (Blocking Prerequisites)

**Purpose**: External setup and Cilium compatibility that MUST be complete before Tailscale Operator deployment

**⚠️ CRITICAL**: No user story work can begin until this phase is complete

- [x] T004 Create OAuth client in Tailscale Admin Console with scopes: `Devices Core` (write), `Auth Keys` (write), `Services` (write) and tags: `tag:k8s-operator`, `tag:k8s`
- [x] T005 Create SOPS-encrypted OAuth secret in `k8s/infrastructure/tailscale/secret-oauth-credentials.sops.yaml`
- [x] T006 [US4] Update Cilium HelmRelease with `socketLB.hostNamespaceOnly: true` in `k8s/infrastructure/cilium/helmrelease.yaml`
- [x] T007 Verify Cilium reconciliation succeeds after socketLB configuration change

**Checkpoint**: Foundation ready - Tailscale Operator can now be deployed

---

## Phase 3: User Story 2 - GitOps-managed Tailscale Operator deployment (Priority: P2)

**Goal**: Deploy Tailscale Operator via Flux with encrypted OAuth credentials

**Independent Test**: After Flux reconciliation, the Tailscale Operator is running and registered as a device in the tailnet with `tag:k8s-operator` tag.

**Note**: US2 is implemented before US1 because the operator must be deployed before any Ingress can be created.

### Implementation for User Story 2

- [x] T008 [US2] Create HelmRelease for Tailscale Operator in `k8s/infrastructure/tailscale/helmrelease.yaml`
- [x] T009 [US2] Create Kustomization entry point in `k8s/infrastructure/tailscale/kustomization.yaml`
- [x] T010 [US2] Add tailscale to infrastructure kustomization in `k8s/infrastructure/kustomization.yaml`
- [x] T011 [US2] Commit and push changes to trigger Flux reconciliation
- [x] T012 [US2] Verify operator pod is running: `kubectl get pods -n tailscale`
- [x] T013 [US2] Verify operator registered in Tailscale Admin Console with `tag:k8s-operator`

**Checkpoint**: Tailscale Operator is deployed and registered in tailnet

---

## Phase 4: User Story 4 - Cilium kube-proxy replacement compatibility (Priority: P1)

**Goal**: Ensure Tailscale proxies work correctly with Cilium's kube-proxy replacement mode

**Independent Test**: Services exposed via Tailscale Ingress correctly route traffic to backend pods despite Cilium's socket-level load balancing.

**Note**: Cilium configuration was done in Phase 2 (T006-T007). This phase validates the integration.

### Verification for User Story 4

- [x] T014 [US4] Verify Cilium pods restarted and healthy after config change: `kubectl get pods -n kube-system -l app.kubernetes.io/name=cilium`
- [x] T015 [US4] Verify socketLB configuration applied: `kubectl get configmap -n kube-system cilium-config -o yaml | grep socketLB`

**Checkpoint**: Cilium is configured for Tailscale compatibility

---

## Phase 5: User Story 3 - Resource-efficient proxy management with ProxyGroup (Priority: P3)

**Goal**: Configure ProxyGroup with 1 replica for resource-efficient shared proxy infrastructure

**Independent Test**: ProxyGroup is ready and creates a single StatefulSet with 1 replica that can serve multiple Ingress resources.

### Implementation for User Story 3

- [x] T016 [US3] Create ProxyGroup manifest in `k8s/infrastructure/tailscale/proxygroup.yaml`
- [x] T017 [US3] Add proxygroup.yaml to kustomization in `k8s/infrastructure/tailscale/kustomization.yaml`
- [x] T018 [US3] Commit and push changes to trigger Flux reconciliation
- [x] T019 [US3] Wait for ProxyGroup ready: `kubectl wait proxygroup ingress-proxies --for=condition=ProxyGroupReady=true -n tailscale --timeout=300s`
- [x] T020 [US3] Verify StatefulSet created with 1 replica: `kubectl get statefulset -n tailscale`

**Checkpoint**: ProxyGroup is ready for Ingress resources

---

## Phase 6: User Story 1 - Expose Kubernetes services to tailnet via Ingress (Priority: P1) 🎯 MVP

**Goal**: Expose Longhorn UI via Tailscale Ingress and verify access from tailnet

**Independent Test**: Longhorn UI is accessible from a tailnet device via `https://longhorn.<tailnet>.ts.net` with valid TLS certificate.

### Implementation for User Story 1

- [x] T021 [US1] Create smoke test directory structure in `k8s/infrastructure/tailscale/smoke/`
- [x] T022 [P] [US1] Create Ingress for Longhorn UI in `k8s/infrastructure/tailscale/smoke/ingress-longhorn.yaml`
- [x] T023 [P] [US1] Create smoke test kustomization in `k8s/infrastructure/tailscale/smoke/kustomization.yaml`
- [x] T024 [US1] Enable smoke test in main kustomization by adding `smoke/` to `k8s/infrastructure/tailscale/kustomization.yaml`
- [x] T025 [US1] Commit and push changes to trigger Flux reconciliation
- [x] T026 [US1] Verify Ingress created: `kubectl get ingress -n longhorn-system longhorn-ui`
- [x] T027 [US1] Access Longhorn UI from tailnet device: `curl -I https://longhorn.<tailnet>.ts.net`
- [x] T028 [US1] Verify TLS certificate is valid and issued by Tailscale
- [x] T029 [US1] Verify service is NOT accessible from public internet

**Checkpoint**: Longhorn UI is accessible via Tailscale Ingress with TLS - MVP Complete!

---

## Phase 7: Polish & Cross-Cutting Concerns

**Purpose**: Documentation and cleanup

- [x] T030 [P] Update docs with Tailscale Operator operational procedures
- [x] T031 [P] Disable smoke test resources (comment out in kustomization) if not needed permanently
- [x] T032 Run quickstart.md validation to ensure documentation matches implementation
- [x] T033 Final commit with all changes

---

## Dependencies & Execution Order

### Phase Dependencies

```
Phase 1 (Setup)
    │
    ▼
Phase 2 (Foundational) ─── BLOCKS ALL USER STORIES
    │
    ├──────────────────────────────┐
    ▼                              ▼
Phase 3 (US2: Operator)      Phase 4 (US4: Cilium - verification only)
    │
    ▼
Phase 5 (US3: ProxyGroup)
    │
    ▼
Phase 6 (US1: Ingress) ─── MVP COMPLETE
    │
    ▼
Phase 7 (Polish)
```

### User Story Dependencies

- **User Story 4 (Cilium)**: Part of Foundational phase - must complete first
- **User Story 2 (Operator)**: Depends on Foundational - operator must be deployed before any Ingress
- **User Story 3 (ProxyGroup)**: Depends on US2 - operator must be running to manage ProxyGroup
- **User Story 1 (Ingress)**: Depends on US2 and US3 - needs operator and ProxyGroup ready

### Within Each Phase

- Tasks marked [P] can run in parallel
- Non-parallel tasks must complete in order
- Verification tasks (kubectl commands) follow implementation tasks

### Parallel Opportunities

- T002, T003 (namespace and helmrepository) can run in parallel
- T022, T023 (ingress and kustomization for smoke test) can run in parallel
- T030, T031 (documentation tasks) can run in parallel

---

## Parallel Example: Phase 1

```bash
# Launch these tasks in parallel:
Task T002: "Create namespace manifest in k8s/infrastructure/tailscale/namespace.yaml"
Task T003: "Create HelmRepository in k8s/infrastructure/tailscale/helmrepository.yaml"
```

## Parallel Example: Phase 6

```bash
# Launch these tasks in parallel:
Task T022: "Create Ingress for Longhorn UI in k8s/infrastructure/tailscale/smoke/ingress-longhorn.yaml"
Task T023: "Create smoke test kustomization in k8s/infrastructure/tailscale/smoke/kustomization.yaml"
```

---

## Implementation Strategy

### MVP First (Through Phase 6)

1. Complete Phase 1: Setup (T001-T003)
2. Complete Phase 2: Foundational (T004-T007) - CRITICAL
3. Complete Phase 3: Deploy Operator (T008-T013)
4. Complete Phase 4: Verify Cilium (T014-T015)
5. Complete Phase 5: ProxyGroup (T016-T020)
6. Complete Phase 6: Ingress (T021-T029)
7. **STOP and VALIDATE**: Access Longhorn UI from tailnet
8. MVP Complete - Tailscale Ingress working!

### Incremental Delivery

1. After Phase 3 → Operator running, can verify in Tailscale console
2. After Phase 5 → ProxyGroup ready, resource-efficient proxy available
3. After Phase 6 → Full Ingress functionality demonstrated with Longhorn UI

---

## Notes

- [P] tasks = different files, no dependencies
- [Story] label maps task to specific user story for traceability
- Manual verification via kubectl and tailnet access (no automated tests)
- Commit after each phase completion
- OAuth credentials are created manually in Tailscale Admin Console (T004)
- Smoke test can be disabled after validation by commenting out in kustomization
