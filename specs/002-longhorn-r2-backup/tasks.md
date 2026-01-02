---

description: "Actionable task list for introducing Longhorn with Cloudflare R2 backups"
---

# Tasks: Introduce Longhorn with R2 Backups

**Input**: Design documents from `/specs/002-longhorn-r2-backup/`
**Prerequisites**: `plan.md` (required), `spec.md` (required), `research.md`, `data-model.md`, `contracts/`, `quickstart.md`
**Tests**: No dedicated automated test suite requested in `spec.md`; tasks include deterministic GitOps validation steps and smoke checks.

## Format: `[ID] [P?] [Story] Description (with file path)`

- **[P]**: Can run in parallel (different files, no dependencies)
- **[Story]**: Which user story this task belongs to: **[US1] [US2] [US3]**

---

## Phase 1: Setup (Shared Infrastructure)

**Purpose**: Add repo wiring needed for Longhorn + SOPS usage in `k8s/`

- [ ] T001 Create `k8s/infrastructure/longhorn/` directory and `k8s/infrastructure/longhorn/kustomization.yaml`
- [ ] T002 Update root SOPS rules to include Kubernetes secrets by adding a `creation_rules` entry for `k8s/.*\\.sops\\.yaml$` in `.sops.yaml`
- [ ] T003 [P] Document the out-of-band bootstrap step for the Flux SOPS age key Secret in `specs/002-longhorn-r2-backup/quickstart.md` (e.g., create `sops-age` Secret in `flux-system` from local `age.agekey`)
- [ ] T004 Update `k8s/infrastructure/kustomization.yaml` to include `longhorn/`

---

## Phase 2: Foundational (Blocking Prerequisites)

**Purpose**: Ensure Flux can decrypt SOPS secrets for `k8s/infrastructure` reconciliation

**⚠️ CRITICAL**: No user story work should be considered complete until decryption + reconciliation is validated.

- [ ] T005 Configure Flux SOPS decryption for infrastructure by updating `k8s/flux/infrastructure-kustomization.yaml` with `spec.decryption.provider: sops` and `spec.decryption.secretRef` to the age key Secret in `flux-system`
- [ ] T006 Add an operator-visible “reconciliation contract” validation checklist section to `specs/002-longhorn-r2-backup/contracts/reconciliation.md` (verify HelmRepository/HelmRelease readiness + decrypted Secret presence)
- [ ] T007 Validate manifests render: run `kustomize build k8s/flux` and `kustomize build k8s/infrastructure` (document command + expected outcome in `specs/002-longhorn-r2-backup/quickstart.md`)

**Checkpoint**: Foundation ready — Flux can decrypt `*.sops.yaml` resources under `k8s/infrastructure/` and reconcile them.

---

## Phase 3: User Story 1 — Reliable single-node persistent storage (Priority: P1) 🎯 MVP

**Goal**: Longhorn installed via Flux and tuned for single-node defaults (replicas=1)

**Independent Test**: After Flux reconciliation, Longhorn is Ready and a PVC-backed test workload can write/read data across restarts.

### Implementation for User Story 1

- [ ] T008 [P] [US1] Add `HelmRepository` for Longhorn chart source in `k8s/infrastructure/longhorn/helmrepository.yaml`
- [ ] T009 [P] [US1] Add namespace manifest for Longhorn (if required by your repo conventions) in `k8s/infrastructure/longhorn/namespace.yaml`
- [ ] T010 [US1] Add `HelmRelease` for Longhorn in `k8s/infrastructure/longhorn/helmrelease.yaml` with default replica count configured to **1** (per `spec.md` Clarifications + FR-002a)
- [ ] T011 [US1] Wire Longhorn resources into `k8s/infrastructure/longhorn/kustomization.yaml` (include repository, release, and any namespace)
- [ ] T012 [US1] Add a minimal “PVC smoke test” manifest set under `k8s/infrastructure/longhorn/smoke/` (PVC + Pod that writes a known value) and include it in `k8s/infrastructure/longhorn/kustomization.yaml` behind an easy toggle (commented resource entry or separate kustomization)
- [ ] T013 [US1] Update `specs/002-longhorn-r2-backup/quickstart.md` with the manual steps to validate SC-001/SC-002 (create test workload, restart pod, verify data)

**Checkpoint**: US1 delivers persistent storage on the single-node cluster without “degraded-by-design” defaults.

---

## Phase 4: User Story 2 — Backup and restore volumes to Cloudflare R2 (Priority: P2)

**Goal**: Daily scheduled backups to Cloudflare R2 with 30-day retention, and restore workflow validated

**Independent Test**: A test volume can be backed up to R2 and restored into a new volume with data integrity verified.

### Implementation for User Story 2

- [ ] T014 [US2] Define Longhorn backup target defaults in `k8s/infrastructure/longhorn/helmrelease.yaml` (backup target = Cloudflare R2, schedule = daily, retention = 30 days)
- [ ] T015 [US2] Add SOPS-encrypted Secret for R2 credentials in `k8s/infrastructure/longhorn/secret-r2-credentials.sops.yaml` (ensure file name matches `*.sops.yaml` convention)
- [ ] T016 [US2] Reference the R2 credential Secret from `k8s/infrastructure/longhorn/helmrelease.yaml` (credential secret name and namespace alignment)
- [ ] T017 [US2] Add restore validation guidance to `specs/002-longhorn-r2-backup/contracts/backup-restore.md` (what signals to check; map to SC-003/SC-004/SC-007)
- [ ] T018 [US2] Update `specs/002-longhorn-r2-backup/quickstart.md` with operator steps to validate backups/restore end-to-end (create known test data, confirm backup, restore to new volume, verify)

**Checkpoint**: US2 provides recoverability via R2 backups (daily) with 30-day retention.

---

## Phase 5: User Story 3 — Encrypted backup credentials managed in Git (Priority: P3)

**Goal**: No plaintext credentials in Git; Flux decrypts SOPS-encrypted secrets during reconciliation reliably

**Independent Test**: Repository contains only encrypted secrets; reconciliation produces usable in-cluster Secret and backups can authenticate to R2.

### Implementation for User Story 3

- [ ] T019 [US3] Extend `.sops.yaml` so that `k8s/**/*.sops.yaml` uses the same age recipients as existing Talos secrets, and verify it does not break existing Talos encryption rules
- [ ] T020 [US3] Add a “no plaintext secrets” verification step to `specs/002-longhorn-r2-backup/quickstart.md` (e.g., repo search patterns; map to SC-005)
- [ ] T021 [US3] Ensure Flux decryption failure modes are documented in `specs/002-longhorn-r2-backup/contracts/reconciliation.md` (missing key, wrong secretRef, invalid SOPS file)
- [ ] T022 [US3] Add a minimal example SOPS-encrypted Secret (non-sensitive demo value) under `k8s/infrastructure/longhorn/examples/` to prove decryption wiring works independently of R2 creds

**Checkpoint**: US3 ensures secret handling complies with constitution and is operationally debuggable.

---

## Final Phase: Polish & Cross-Cutting Concerns

**Purpose**: Make the feature maintainable and reviewable

- [ ] T023 [P] Normalize file naming and ensure all encrypted secrets follow `*.sops.yaml` naming across `k8s/`
- [ ] T024 Update `README.md` or `docs/` to mention Longhorn as the storage provider and point to `specs/002-longhorn-r2-backup/quickstart.md`
- [ ] T025 Ensure `specs/002-longhorn-r2-backup/` docs are consistent (plan/spec/research/contracts/quickstart): no contradictions about replicas, schedule, retention

---

## Dependencies & Execution Order

### Phase Dependencies

- **Setup (Phase 1)**: no dependencies
- **Foundational (Phase 2)**: depends on Setup; **blocks** all user stories
- **US1/US2/US3**: can start after Foundational completes; implement sequentially by priority (P1 → P2 → P3) unless you have parallel capacity
- **Polish**: after desired user stories are complete

### User Story Dependencies

- **US1 (P1)**: requires decryption wiring to be in place (Phase 2), then can proceed independently
- **US2 (P2)**: depends on US1 (Longhorn installed) and decryption wiring (for credentials)
- **US3 (P3)**: depends on Phase 2 decryption wiring; otherwise independent of US1/US2 design-wise, but validates the secrets posture for US2 credentials

### Parallel Opportunities

- Setup tasks marked **[P]** can run in parallel (docs vs YAML)
- Within US1, `helmrepository.yaml` and `namespace.yaml` can be authored in parallel (**T008/T009**)

---

## Parallel Example: US1

```text
Task: T008 [US1] Add HelmRepository in k8s/infrastructure/longhorn/helmrepository.yaml
Task: T009 [US1] Add namespace manifest in k8s/infrastructure/longhorn/namespace.yaml
```

---

## MVP Scope

- **MVP = User Story 1 (US1)**: Flux-managed Longhorn installed + single-node tuning + PVC smoke validation.
