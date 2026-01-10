# Implementation Plan: Deploy Vikunja

**Branch**: `014-deploy-vikunja` | **Date**: 2026-01-10 | **Spec**: [spec.md](./spec.md)  
**Input**: Feature specification from `/specs/014-deploy-vikunja/spec.md`

## Summary

Deploy Vikunja as a GitOps-managed Kubernetes application, accessible only from the trusted network/VPN via the existing Tailscale Ingress pattern. Use the existing CloudNativePG PostgreSQL cluster for durable data storage with daily automated backups to R2, and use a Longhorn-backed PVC for file/attachment storage with daily volume backups. Configure Vikunja to **disable self-registration**, use **local accounts only** (no SSO), and support collaborative shared projects (shared users can edit).

## Technical Context

**Language/Version**: Kubernetes manifests (YAML), Flux CD + Kustomize  
**Primary Dependencies**: Flux CD, Kustomize, SOPS + age, Tailscale Ingress, CloudNativePG, Longhorn, Vikunja Helm chart  
**Storage**:
- PostgreSQL via CloudNativePG (`k8s/configs/postgres/`) with ScheduledBackup to R2
- Longhorn PVC for Vikunja file storage (attachments)
**Testing**: `kustomize build`, Flux reconciliation checks, pod readiness, browser smoke tests, backup/restore dry-runs  
**Target Platform**: Talos Linux Kubernetes (homelab)  
**Project Type**: GitOps Kubernetes application deployment  
**Performance Goals**: Interactive UI usable immediately after reconciliation; basic flows (sign-in + create task) complete within 2 minutes on home network (per spec SC-001)  
**Constraints**:
- No direct public internet exposure (Tailscale/trusted network only)
- No public self sign-up (admin-created users only)
- No SSO for MVP (local accounts only)
- Daily backups for **all** user data (database + files)
**Scale/Scope**: Household-scale usage (single cluster, small user count, single-node topology acceptable)

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-check after Phase 1 design.*

| Principle | Status | Evidence |
|-----------|--------|----------|
| I. GitOps-First | ✅ Pass | All resources defined under `k8s/` and reconciled by Flux |
| II. Declarative Configuration | ✅ Pass | Kustomize + HelmRelease resources, no long-lived imperative drift |
| III. Secrets Management | ✅ Pass | Secrets stored as SOPS-encrypted `*.sops.yaml` and decrypted by Flux |
| IV. Immutable Infrastructure | ✅ Pass | No node-level mutable changes; everything in-cluster |
| V. Documentation as Code | ✅ Pass | Plan + research + contracts + quickstart under `specs/014-deploy-vikunja/` |

## Project Structure

### Documentation (this feature)

```text
specs/014-deploy-vikunja/
├── plan.md                     # This file
├── research.md                  # Phase 0 output
├── data-model.md                # Phase 1 output
├── quickstart.md                # Phase 1 output
├── contracts/                   # Phase 1 output
│   ├── reconciliation.md
│   └── backup-restore.md
└── tasks.md                     # Phase 2 output (created by /speckit.tasks)
```

### Source Code (repository root)

```text
k8s/
├── apps/
│   ├── kustomization.yaml
│   └── vikunja/
│       ├── namespace.yaml
│       ├── kustomization.yaml
│       └── app/
│           ├── kustomization.yaml
│           ├── helmrepository.yaml              # Vikunja chart repo (Flux source)
│           ├── helmrelease.yaml                 # Vikunja chart (configured for external DB, tailscale ingress)
│           ├── pvc.yaml                         # Longhorn PVC for file storage (+ daily backup annotation)
│           ├── ingress.yaml                     # Tailscale ingress (if chart ingress is not used)
│           ├── secret-vikunja.sops.yaml         # JWT secret and/or app config (SOPS)
│           └── secret-db.sops.yaml              # DB connection info (SOPS) (if not reusing postgres secret)
│
└── configs/
    └── postgres/
        ├── cluster.yaml                         # Existing CNPG cluster (extended for role management)
        ├── scheduledbackup.yaml                 # Existing daily backups
        ├── database-vikunja.yaml                # Database CRD for vikunja
        └── secret-vikunja-db.sops.yaml          # Role passwordSecret (SOPS)
```

**Structure Decision**:
- Follow existing application layout under `k8s/apps/<app>/` with Flux + Kustomize.
- Use the official Vikunja Helm chart via a Flux `HelmRelease` to avoid maintaining low-level manifests.
- Reuse the existing CloudNativePG cluster and manage the Vikunja database/role declaratively (CNPG `managed.roles` + `Database` CRD).
- Use a Longhorn PVC for Vikunja file storage and enable daily backups via PVC annotation (workaround for StorageClass recurring job selector).

## Implementation Phases

### Phase 0: Research & Decisions (output: `research.md`)

1. Confirm Vikunja chart repository + key configuration knobs (external DB, persistence, ingress).
2. Confirm how to disable user registration in Vikunja and how to create users/admin accounts.
3. Confirm CNPG declarative role + database management approach (`managed.roles` + `Database` CRD) for application credentials.
4. Define backup coverage: DB (CNPG ScheduledBackup) + files (Longhorn recurring backup via PVC annotation).

### Phase 1: Design & Contracts (outputs: `data-model.md`, `contracts/*`, `quickstart.md`)

**Design decisions**:
- DB credentials stored in SOPS-encrypted Secret referenced by CloudNativePG for role password management.
- Vikunja connects to Postgres via in-cluster service DNS and reads sensitive values (DB password, JWT secret) from Secrets.
- Vikunja file storage backed by Longhorn PVC with daily recurring backup.
- Exposure limited to Tailscale ingress (host `vikunja`) and TLS handled by the existing ingress pattern.

**Deliverables**:
- Data model for resources and invariants.
- Reconciliation contract (what “healthy” looks like, and what signals to check).
- Backup/restore contract (what is backed up, how to validate, how to restore).
- Quickstart steps for first-time provisioning (including admin user creation).

### Phase 2: Execution Planning (tasks created by `/speckit.tasks`)

Break the work into reviewable GitOps steps:
- Add app namespace + kustomizations
- Add DB role/database resources + encrypted secrets
- Add HelmRepository/HelmRelease + persistence/ingress wiring
- Validate reconciliation and user flows
- Validate backup + restore procedures

## Complexity Tracking

No constitution violations requiring justification. This plan follows existing repository patterns (Flux + Kustomize + SOPS) and reuses shared infrastructure (CNPG + Longhorn + Tailscale ingress).
