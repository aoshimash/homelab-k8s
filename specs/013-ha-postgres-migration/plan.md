# Implementation Plan: Home Assistant DB Migration to PostgreSQL Cluster

**Branch**: `013-ha-postgres-migration` | **Date**: 2026-01-09 | **Spec**: [spec.md](./spec.md)
**Input**: Feature specification from `/specs/013-ha-postgres-migration/spec.md`

## Summary

Migrate Home Assistant's Recorder database from the local SQLite file in the `/config` PVC to the existing CloudNativePG PostgreSQL cluster, preserving **all** history with a planned downtime of **≤ 30 minutes**. Use a one-off migration Job (pgloader) during cutover, keep rollback possible for **24 hours**, and configure Recorder retry settings to reduce impact when the database is temporarily unavailable.

## Technical Context

**Language/Version**: YAML (Kubernetes manifests), Home Assistant 2024.12  
**Primary Dependencies**: Flux CD, Kustomize, CloudNativePG, pgloader  
**Storage**: Longhorn PVC (`/config`) + PostgreSQL (CloudNativePG)  
**Testing**: Manual verification via Home Assistant UI + `kubectl` logs/rollouts + basic SQL sanity checks  
**Target Platform**: Kubernetes on Talos Linux  
**Project Type**: GitOps-managed Kubernetes application + shared database config  
**Performance Goals**: Planned downtime ≤ 30 minutes; UI returns to healthy state immediately after cutover  
**Constraints**:
- Planned downtime (Home Assistant unavailable) **≤ 30 minutes**
- Preserve **all** history (no intentional truncation)
- Rollback must remain possible for **≥ 24 hours**
- Declarative changes only (GitOps-first), no long-lived imperative drift

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-check after Phase 1 design.*

| Principle | Status | Evidence |
|-----------|--------|----------|
| I. GitOps-First | ✅ Pass | All changes in `k8s/` and `specs/` committed, Flux reconciles |
| II. Declarative Configuration | ✅ Pass | Kustomize-managed YAML manifests |
| III. Secrets Management | ✅ Pass | Database connection information stored in SOPS-encrypted Secret |
| IV. Immutable Infrastructure | ✅ Pass | No node-level changes; migration executed as in-cluster Job |
| V. Documentation as Code | ✅ Pass | Plan + runbooks under `specs/013-ha-postgres-migration/` |

## Project Structure

### Documentation (this feature)

```text
specs/013-ha-postgres-migration/
├── plan.md                     # This file
├── research.md                  # Phase 0 output
├── data-model.md                # Phase 1 output
├── quickstart.md                # Phase 1 output
├── contracts/                   # Phase 1 output
│   ├── reconciliation.md
│   ├── backup-restore.md
│   └── cutover.md
└── tasks.md                     # Phase 2 output (created by /speckit.tasks)
```

### Source Code (repository root)

```text
k8s/
├── apps/
│   └── home-assistant/
│       ├── namespace.yaml
│       ├── kustomization.yaml
│       └── app/
│           ├── deployment.yaml
│           ├── pvc.yaml
│           ├── service.yaml
│           ├── ingress.yaml
│           ├── configmap.yaml
│           ├── kustomization.yaml
│           ├── secret-db-url.sops.yaml          # NEW (DB_URL)
│           └── configuration.yaml               # NEW (tracked HA config; db_url uses !env_var)
│
└── configs/
    └── postgres/                                # Existing CloudNativePG cluster
        ├── namespace.yaml
        ├── cluster.yaml
        ├── scheduledbackup.yaml
        └── secret-r2-credentials.sops.yaml
```

**Structure Decision**:
- Keep Home Assistant changes scoped under `k8s/apps/home-assistant/`.
- Reuse the existing shared PostgreSQL cluster in `k8s/configs/postgres/`.
- Treat the migration Job as a short-lived operational tool (applied via Git, then removed in a follow-up change after successful cutover).

## Implementation Phases

### Phase 0: Research & Decisions (output: `research.md`)

1. Confirm Home Assistant Recorder supports PostgreSQL via `recorder.db_url`.
2. Select a migration method that can preserve all history within the downtime constraint.
3. Define how Home Assistant should behave when DB connectivity is temporarily lost (retry settings).
4. Define rollback boundaries and verification signals.

### Phase 1: Design & Contracts (outputs: `data-model.md`, `contracts/*`, `quickstart.md`)

**Design decisions**:
- Home Assistant will read the database URL from an environment variable (e.g., `DB_URL`) referenced in `configuration.yaml` via `!env_var`.
- The `DB_URL` value is stored in a SOPS-encrypted Secret applied by Flux.
- A one-time Kubernetes Job runs pgloader against the SQLite DB file on the Home Assistant PVC while Home Assistant is stopped.

**Deliverables**:
- Resource-level design for Secrets, config, and migration Job.
- Reconciliation expectations and rollback runbook.
- Quickstart steps to run the migration safely in GitOps flow.

### Phase 2: Execution Planning (tasks created by `/speckit.tasks`)

Break the implementation into safe, reviewable steps:
- Prepare DB/user and Secret
- Prepare config changes (Recorder + retry settings)
- Pre-flight checks (backup/snapshot + estimated migration time)
- Cutover and verification
- 24h observation window and cleanup

## Complexity Tracking

No constitution violations requiring justification. The plan follows existing repository patterns (Flux + Kustomize + SOPS).
