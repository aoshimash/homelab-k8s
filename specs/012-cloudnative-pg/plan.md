# Implementation Plan: CloudNativePG PostgreSQL Cluster

**Branch**: `012-cloudnative-pg` | **Date**: 2026-01-07 | **Spec**: [spec.md](./spec.md)
**Input**: Feature specification from `/specs/012-cloudnative-pg/spec.md`

## Summary

Deploy a shared PostgreSQL cluster using CloudNativePG operator for homelab multi-tenant database needs. The cluster uses PostgreSQL 16 with 10GB Longhorn storage, daily backups to R2 with 7-day retention, and follows existing Flux GitOps patterns.

## Technical Context

**Platform**: Kubernetes (Talos Linux)  
**GitOps**: Flux CD with Kustomize  
**Operator**: CloudNativePG (Helm chart from `https://cloudnative-pg.github.io/charts`)  
**Storage**: Longhorn (existing, configured)  
**Secrets**: SOPS with Age encryption  
**Backup Target**: Cloudflare R2 (S3-compatible, existing Longhorn R2 credentials available)  
**Database Version**: PostgreSQL 16  
**Instances**: 1 (single-node homelab deployment)  
**Storage Size**: 10GB PVC  
**Backup Schedule**: Daily (cron: `0 0 0 * * *`)  
**Backup Retention**: 7 days

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-check after Phase 1 design.*

| Principle | Status | Notes |
|-----------|--------|-------|
| I. GitOps-First | ✅ Pass | All manifests committed to Git, Flux reconciles |
| II. Declarative Configuration | ✅ Pass | HelmRelease + Cluster CRD, Kustomize structure |
| III. Secrets Management | ✅ Pass | SOPS-encrypted secrets for DB credentials and R2 access |
| IV. Immutable Infrastructure | ✅ Pass | No node modifications required |
| V. Documentation as Code | ✅ Pass | docs/cloudnative-pg.md to be created |

## Project Structure

### Documentation (this feature)

```text
specs/012-cloudnative-pg/
├── plan.md              # This file
├── research.md          # Phase 0 output
├── data-model.md        # Phase 1 output
├── quickstart.md        # Phase 1 output
├── contracts/           # Phase 1 output
│   └── reconciliation.md
└── tasks.md             # Phase 2 output (created by /speckit.tasks)
```

### Source Code (repository root)

```text
k8s/
├── infrastructure/
│   └── cloudnative-pg/           # Operator installation
│       ├── namespace.yaml        # cnpg-system namespace
│       ├── helmrepository.yaml   # CloudNativePG Helm repo
│       ├── helmrelease.yaml      # Operator HelmRelease
│       └── kustomization.yaml    # Kustomize resources
└── configs/
    └── postgres/                 # PostgreSQL cluster definition
        ├── namespace.yaml        # postgres namespace
        ├── cluster.yaml          # Cluster CRD
        ├── scheduledbackup.yaml  # ScheduledBackup CRD
        ├── secret-r2-credentials.sops.yaml  # R2 backup credentials
        └── kustomization.yaml    # Kustomize resources

docs/
└── cloudnative-pg.md             # Operational documentation
```

**Structure Decision**: Following existing repository patterns:
- Operator in `k8s/infrastructure/cloudnative-pg/` (similar to longhorn, tailscale)
- Cluster definition in `k8s/configs/postgres/` (similar to tailscale proxy configs)
- This separation allows operator upgrades independent of cluster configuration

## Implementation Phases

### Phase 1: Operator Installation

Deploy CloudNativePG operator via Flux HelmRelease:
1. Create `cnpg-system` namespace
2. Add CloudNativePG HelmRepository to flux-system
3. Create HelmRelease for operator
4. Add to infrastructure kustomization

### Phase 2: PostgreSQL Cluster

Deploy PostgreSQL cluster with backup configuration:
1. Create `postgres` namespace
2. Create R2 credentials secret (SOPS encrypted)
3. Create Cluster CRD with storage and backup config
4. Create ScheduledBackup CRD for daily backups
5. Add to configs kustomization

### Phase 3: Documentation

Create operational documentation:
1. Bootstrap procedure
2. Connection instructions
3. Backup/restore procedures
4. Troubleshooting guide

## Complexity Tracking

No constitution violations requiring justification.
