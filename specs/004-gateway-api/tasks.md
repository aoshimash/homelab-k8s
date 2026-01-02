# Tasks: Gateway API CRD Installation

**Input**: Design documents from `/specs/004-gateway-api/`
**Prerequisites**: plan.md (required), spec.md (required for user stories), research.md, data-model.md, contracts/

**Tests**: Verification commands are included as part of implementation (kubectl commands, not automated tests).

**Organization**: Tasks are grouped by user story to enable independent implementation and testing of each story.

## Format: `[ID] [P?] [Story] Description`

- **[P]**: Can run in parallel (different files, no dependencies)
- **[Story]**: Which user story this task belongs to (e.g., US1, US2, US3)
- Include exact file paths in descriptions

## Path Conventions

- **Infrastructure**: `k8s/infrastructure/` directory
- **Gateway API**: `k8s/infrastructure/gateway-api/` (new directory)

---

## Phase 1: Setup (Directory Structure)

**Purpose**: Create the gateway-api directory structure

- [ ] T001 Create gateway-api directory at `k8s/infrastructure/gateway-api/`

---

## Phase 2: User Story 1 - Install Gateway API CRDs (Priority: P1) 🎯 MVP

**Goal**: Install Gateway API CRDs into the cluster so Gateway and HTTPRoute resources can be created

**Independent Test**: Run `kubectl get crd | grep gateway.networking.k8s.io` and verify 5 CRDs are listed

### Implementation for User Story 1

- [ ] T002 [P] [US1] Create GitRepository resource in `k8s/infrastructure/gateway-api/gitrepository.yaml` referencing kubernetes-sigs/gateway-api v1.4.1
- [ ] T003 [P] [US1] Create Flux Kustomization resource in `k8s/infrastructure/gateway-api/kustomization-crds.yaml` with path `./config/crd/standard` and `prune: false`
- [ ] T004 [US1] Create local Kustomization in `k8s/infrastructure/gateway-api/kustomization.yaml` including gitrepository.yaml and kustomization-crds.yaml

**Checkpoint**: Gateway API Flux resources are ready for deployment

---

## Phase 3: User Story 2 - Manage Gateway API via GitOps (Priority: P2)

**Goal**: Integrate Gateway API CRD installation with existing Flux infrastructure

**Independent Test**: Run `kubectl get kustomization gateway-api-crds -n flux-system` and verify Ready=True status

### Implementation for User Story 2

- [ ] T005 [US2] Update infrastructure kustomization at `k8s/infrastructure/kustomization.yaml` to add `gateway-api/` as the first entry in resources list
- [ ] T006 [US2] Commit changes to Git repository with message "feat: add Gateway API CRD installation via Flux"
- [ ] T007 [US2] Push changes to remote repository to trigger Flux reconciliation

**Checkpoint**: Changes committed and pushed, Flux reconciliation triggered

---

## Phase 4: User Story 3 - Enable Longhorn UI Access via Tailnet (Priority: P2)

**Goal**: Verify Gateway API CRDs enable existing Tailscale Gateway/HTTPRoute to function

**Independent Test**: Access `https://longhorn.<tailnet>.ts.net` from a tailnet-connected device

### Verification for User Story 3

- [ ] T008 [US3] Verify GitRepository status with `kubectl get gitrepository gateway-api -n flux-system`
- [ ] T009 [US3] Verify Kustomization status with `kubectl get kustomization gateway-api-crds -n flux-system`
- [ ] T010 [US3] Verify CRDs are installed with `kubectl get crd | grep gateway.networking.k8s.io`
- [ ] T011 [US3] Verify Gateway status with `kubectl get gateway tailscale-gateway -n tailscale -o jsonpath='{.status.conditions}'`
- [ ] T012 [US3] Verify HTTPRoute status with `kubectl get httproute longhorn-ui -n longhorn-system -o jsonpath='{.status.parents[0].conditions}'`
- [ ] T013 [US3] Test Longhorn UI access via browser at `https://longhorn.<tailnet>.ts.net`

**Checkpoint**: All verifications pass, Longhorn UI accessible via tailnet

---

## Phase 5: Polish & Documentation

**Purpose**: Final cleanup and documentation updates

- [ ] T014 Commit final verification results and any adjustments
- [ ] T015 Update spec status from Draft to Complete in `specs/004-gateway-api/spec.md`

---

## Dependencies & Execution Order

### Phase Dependencies

- **Setup (Phase 1)**: No dependencies - can start immediately
- **User Story 1 (Phase 2)**: Depends on Setup completion
- **User Story 2 (Phase 3)**: Depends on User Story 1 completion
- **User Story 3 (Phase 4)**: Depends on User Story 2 completion (Git push triggers Flux)
- **Polish (Phase 5)**: Depends on all user stories being verified

### User Story Dependencies

- **User Story 1 (P1)**: Create Flux resources - No external dependencies
- **User Story 2 (P2)**: Integrate with infrastructure - Depends on US1 resources existing
- **User Story 3 (P2)**: Verify functionality - Depends on US2 being deployed via Flux

### Within Each User Story

- T002 and T003 can run in parallel (different files)
- T004 depends on T002 and T003 (references both files)
- T005 depends on T004 (needs gateway-api directory to exist)
- T006-T007 are sequential (commit then push)
- T008-T013 can be run sequentially as verification steps

### Parallel Opportunities

- T002 and T003 can be created in parallel (both are independent YAML files)
- T008-T012 verification commands can be scripted to run together

---

## Parallel Example: User Story 1

```bash
# Launch parallel file creation:
# File 1: k8s/infrastructure/gateway-api/gitrepository.yaml
# File 2: k8s/infrastructure/gateway-api/kustomization-crds.yaml

# Then create the kustomization.yaml that references both
```

---

## Implementation Strategy

### MVP First (User Story 1 + 2)

1. Complete Phase 1: Setup (create directory)
2. Complete Phase 2: User Story 1 (create Flux resources)
3. Complete Phase 3: User Story 2 (integrate and deploy)
4. **STOP and VALIDATE**: Flux reconciliation starts
5. Proceed to Phase 4: User Story 3 (verification)

### Expected Timeline

| Phase | Tasks | Estimated Time |
|-------|-------|----------------|
| Setup | T001 | 1 minute |
| US1 | T002-T004 | 5 minutes |
| US2 | T005-T007 | 5 minutes |
| US3 | T008-T013 | 10 minutes (includes Flux reconciliation wait) |
| Polish | T014-T015 | 2 minutes |
| **Total** | 15 tasks | ~25 minutes |

---

## File Reference

### Files to Create

| File | Task | Content |
|------|------|---------|
| `k8s/infrastructure/gateway-api/gitrepository.yaml` | T002 | Flux GitRepository referencing kubernetes-sigs/gateway-api v1.4.1 |
| `k8s/infrastructure/gateway-api/kustomization-crds.yaml` | T003 | Flux Kustomization for CRD installation |
| `k8s/infrastructure/gateway-api/kustomization.yaml` | T004 | Local Kustomize config |

### Files to Modify

| File | Task | Change |
|------|------|--------|
| `k8s/infrastructure/kustomization.yaml` | T005 | Add `gateway-api/` as first resource entry |

---

## Notes

- [P] tasks = different files, no dependencies
- [Story] label maps task to specific user story for traceability
- Verification tasks (T008-T013) use kubectl commands, not automated tests
- Flux reconciliation may take 1-2 minutes after push
- If verification fails, check Flux logs: `flux logs --kind=Kustomization --name=infrastructure`
