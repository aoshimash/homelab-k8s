# Tasks: Tailscale Kubernetes Operator Installation

**Input**: Design documents from `/specs/003-tailscale-k8s-operator/`
**Prerequisites**: plan.md, spec.md, research.md, data-model.md, contracts/reconciliation.md

**Tests**: Manual verification via kubectl and Tailscale admin console (no automated tests)

**Organization**: Tasks are grouped by user story to enable independent implementation and testing of each story.

## Format: `[ID] [P?] [Story] Description`

- **[P]**: Can run in parallel (different files, no dependencies)
- **[Story]**: Which user story this task belongs to (e.g., US1, US2, US3)
- Include exact file paths in descriptions

## Path Conventions

- **Infrastructure manifests**: `k8s/infrastructure/tailscale/`
- **Flux configuration**: `k8s/flux/`
- Paths follow existing patterns from `cilium/` and `longhorn/` directories

---

## Phase 1: Setup (Directory Structure)

**Purpose**: Create directory structure and base files

- [x] T001 Create tailscale directory at `k8s/infrastructure/tailscale/`
- [x] T002 [P] Create namespace manifest at `k8s/infrastructure/tailscale/namespace.yaml`
- [x] T003 [P] Create HelmRepository manifest at `k8s/infrastructure/tailscale/helmrepository.yaml`

---

## Phase 2: Foundational (OAuth Credentials)

**Purpose**: Prepare OAuth credentials - MUST be complete before operator deployment

**⚠️ CRITICAL**: Operator cannot start without valid OAuth credentials

- [x] T004 Create OAuth client in Tailscale admin console (manual step - document in quickstart.md)
- [x] T005 Create plain text secret template at `/tmp/secret-oauth.yaml` with OAuth credentials
- [x] T006 Encrypt secret with SOPS and save to `k8s/infrastructure/tailscale/secret-oauth.sops.yaml`
- [x] T007 Verify encrypted file contains `ENC[AES256_GCM` markers (no plain text credentials)

**Checkpoint**: OAuth credentials encrypted and ready for deployment

---

## Phase 3: User Story 1 - Install Tailscale Operator (Priority: P1) 🎯 MVP

**Goal**: Install Tailscale Operator via Helm and verify it joins the tailnet

**Independent Test**:
- Operator Pod is `Running` in `tailscale` namespace
- Device `tailscale-operator` appears in Tailscale admin console with `tag:k8s-operator`

### Implementation for User Story 1

- [x] T008 [US1] Create HelmRelease manifest at `k8s/infrastructure/tailscale/helmrelease.yaml`
- [x] T009 [US1] Configure HelmRelease to reference OAuth secret via `valuesFrom`
- [x] T010 [US1] Pin Helm chart version in HelmRelease spec (check latest stable version)
- [x] T011 [US1] Create Kustomization at `k8s/infrastructure/tailscale/kustomization.yaml`
- [x] T012 [US1] Update `k8s/infrastructure/kustomization.yaml` to include `tailscale/`

### Verification for User Story 1

- [ ] T013 [US1] Commit and push changes to trigger Flux reconciliation
- [ ] T014 [US1] Verify HelmRepository status: `kubectl get helmrepository tailscale -n flux-system`
- [ ] T015 [US1] Verify HelmRelease status: `kubectl get helmrelease tailscale-operator -n tailscale`
- [ ] T016 [US1] Verify operator Pod: `kubectl get pods -n tailscale`
- [ ] T017 [US1] Verify device in Tailscale admin console with correct hostname and tags

**Checkpoint**: Tailscale Operator running and connected to tailnet - MVP complete

---

## Phase 4: User Story 2 - GitOps Management (Priority: P2)

**Goal**: Verify Flux manages the operator lifecycle automatically

**Independent Test**:
- HelmRelease shows `Ready=True`
- Changes to HelmRelease in Git trigger automatic reconciliation

### Implementation for User Story 2

- [ ] T018 [US2] Verify Flux Kustomization includes Tailscale resources: `flux get kustomization infrastructure`
- [ ] T019 [US2] Test reconciliation by modifying a non-critical value and pushing to Git
- [ ] T020 [US2] Verify automatic reconciliation: `flux get helmrelease tailscale-operator -n tailscale`

**Checkpoint**: GitOps workflow verified - operator managed declaratively

---

## Phase 5: User Story 3 - Secure Credential Management (Priority: P2)

**Goal**: Verify OAuth credentials are encrypted and Flux can decrypt them

**Independent Test**:
- Raw file in repository shows `ENC[AES256_GCM` (encrypted)
- Flux decrypts and creates Secret successfully
- Operator logs show no authentication errors

### Verification for User Story 3

- [ ] T021 [US3] Verify encrypted file: `cat k8s/infrastructure/tailscale/secret-oauth.sops.yaml | grep -q "ENC\["`
- [ ] T022 [US3] Verify Secret created in cluster: `kubectl get secret tailscale-operator-oauth -n tailscale`
- [ ] T023 [US3] Verify no auth errors in operator logs: `kubectl logs -n tailscale -l app.kubernetes.io/name=tailscale-operator | grep -i auth`

**Checkpoint**: Credentials securely managed via SOPS

---

## Phase 6: Ingress Validation (Longhorn UI)

**Goal**: Expose Longhorn Web UI via Tailscale Ingress as validation

**Independent Test**:
- Ingress has hostname assigned
- Longhorn UI accessible via `https://longhorn.<tailnet>.ts.net`

### Implementation for Ingress Validation

- [ ] T024 [P] Create Ingress manifest at `k8s/infrastructure/tailscale/ingress-longhorn.yaml`
- [ ] T025 Update Kustomization to include `ingress-longhorn.yaml`
- [ ] T026 Commit and push changes

### Verification for Ingress

- [ ] T027 Verify Ingress status: `kubectl get ingress longhorn-ui -n longhorn-system`
- [ ] T028 Verify Tailscale proxy created: `kubectl get statefulset -n tailscale`
- [ ] T029 Verify device in Tailscale admin console for Longhorn Ingress
- [ ] T030 Access Longhorn UI via tailnet hostname from a device on the tailnet

**Checkpoint**: Longhorn UI accessible via Tailscale - full feature validation complete

---

## Phase 7: Polish & Documentation

**Purpose**: Final cleanup and documentation updates

- [ ] T031 [P] Update `specs/003-tailscale-k8s-operator/quickstart.md` with actual version numbers used
- [ ] T032 [P] Verify all manifests follow existing patterns (compare with `cilium/` and `longhorn/`)
- [ ] T033 Run full validation per `contracts/reconciliation.md` success criteria mapping
- [ ] T034 Create summary commit with all changes

---

## Dependencies & Execution Order

### Phase Dependencies

- **Setup (Phase 1)**: No dependencies - can start immediately
- **Foundational (Phase 2)**: Depends on Setup - BLOCKS operator deployment
- **User Story 1 (Phase 3)**: Depends on Foundational - Core operator installation
- **User Story 2 (Phase 4)**: Depends on User Story 1 - GitOps verification
- **User Story 3 (Phase 5)**: Depends on User Story 1 - Credential verification
- **Ingress Validation (Phase 6)**: Depends on User Story 1 - Ingress feature
- **Polish (Phase 7)**: Depends on all phases complete

### User Story Dependencies

```
Phase 1: Setup
    │
    ▼
Phase 2: Foundational (OAuth)
    │
    ▼
Phase 3: User Story 1 (Install Operator) ─── MVP COMPLETE
    │
    ├──▶ Phase 4: User Story 2 (GitOps)
    │
    ├──▶ Phase 5: User Story 3 (Credentials)
    │
    └──▶ Phase 6: Ingress Validation
              │
              ▼
         Phase 7: Polish
```

### Parallel Opportunities

**Phase 1 (Setup)**:
- T002 and T003 can run in parallel (different files)

**Phase 3 (User Story 1)**:
- T008-T012 are sequential (file dependencies within Kustomization)

**Phase 6 (Ingress)**:
- T024 can start while T025-T026 wait

**Post-MVP (after Phase 3)**:
- Phases 4, 5, and 6 can run in parallel (independent verification paths)

---

## Parallel Example: Setup Phase

```bash
# Launch these tasks together:
Task T002: "Create namespace manifest at k8s/infrastructure/tailscale/namespace.yaml"
Task T003: "Create HelmRepository manifest at k8s/infrastructure/tailscale/helmrepository.yaml"
```

---

## Implementation Strategy

### MVP First (User Story 1 Only)

1. Complete Phase 1: Setup (T001-T003)
2. Complete Phase 2: Foundational OAuth (T004-T007)
3. Complete Phase 3: User Story 1 (T008-T017)
4. **STOP and VALIDATE**: Operator running, device in admin console
5. Deploy/demo if ready - MVP achieved!

### Incremental Delivery

1. Setup + Foundational → Credentials ready
2. User Story 1 → Operator running → **MVP!**
3. User Story 2 → GitOps verified
4. User Story 3 → Security verified
5. Ingress Validation → Longhorn UI accessible
6. Polish → Documentation complete

### Single Developer Strategy

Execute phases sequentially:
1. Phase 1 → Phase 2 → Phase 3 (MVP)
2. Phase 4 → Phase 5 → Phase 6 (enhancements)
3. Phase 7 (polish)

---

## Notes

- [P] tasks = different files, no dependencies
- [Story] label maps task to specific user story for traceability
- Manual verification via kubectl and Tailscale admin console
- Commit after each phase completion
- OAuth client creation (T004) is a manual prerequisite in Tailscale admin console
- Stop at any checkpoint to validate independently
