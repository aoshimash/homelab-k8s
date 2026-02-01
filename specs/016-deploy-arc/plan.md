# Implementation Plan: Deploy Actions Runner Controller (ARC)

**Branch**: `016-deploy-arc` | **Date**: 2026-02-01 | **Spec**: [spec.md](./spec.md)
**Input**: Feature specification from `/specs/016-deploy-arc/spec.md`

## Summary

Deploy Actions Runner Controller (ARC) to the homelab Kubernetes cluster to enable self-hosted GitHub Actions runners. The deployment uses the modern "runner scale sets" mode with GitHub App authentication, managed via FluxCD GitOps workflow. Runners will be registered at the organization level with the label `homelab`, scaling from 0-3 pods with 4 CPU / 8GB RAM each.

## Technical Context

**Language/Version**: YAML (Kubernetes manifests), Helm charts via FluxCD HelmRelease  
**Primary Dependencies**: 
- ARC Controller: `ghcr.io/actions/actions-runner-controller-charts/gha-runner-scale-set-controller`
- Runner Scale Set: `ghcr.io/actions/actions-runner-controller-charts/gha-runner-scale-set`
**Storage**: N/A (ephemeral runners, no persistent storage required)  
**Testing**: Manual workflow trigger verification, kubectl pod status checks  
**Target Platform**: Homelab Kubernetes cluster (Talos Linux)  
**Project Type**: Infrastructure (Kubernetes GitOps manifests)  
**Performance Goals**: Runner pods ready within 2 minutes of workflow trigger  
**Constraints**: Max 3 concurrent runners (12 CPU, 24GB RAM total), organization-level registration  
**Scale/Scope**: Single organization, all repositories within org can use runners

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-check after Phase 1 design.*

| Principle | Status | Evidence |
|-----------|--------|----------|
| I. GitOps-First | ✅ PASS | Deployment via FluxCD HelmRelease, no manual helm install commands |
| II. Declarative Configuration | ✅ PASS | All config in YAML manifests, Kustomize for structure |
| III. Secrets Management | ✅ PASS | GitHub App credentials stored as SOPS-encrypted secrets |
| IV. Immutable Infrastructure | ✅ PASS | No node-level changes required, runs as standard K8s workload |
| V. Documentation as Code | ✅ PASS | Spec and plan documented in specs/ directory |

**Gate Result**: PASS - All principles satisfied, proceed to Phase 0.

## Project Structure

### Documentation (this feature)

```text
specs/016-deploy-arc/
├── plan.md              # This file
├── research.md          # Phase 0 output
├── data-model.md        # Phase 1 output (K8s resource model)
├── quickstart.md        # Phase 1 output (deployment guide)
└── tasks.md             # Phase 2 output (/speckit.tasks command)
```

### Source Code (repository root)

```text
k8s/infrastructure/actions-runner-controller/
├── namespace.yaml                    # arc-systems namespace for controller
├── helmrepository.yaml              # OCI HelmRepository for ARC charts
├── helmrelease-controller.yaml      # HelmRelease for gha-runner-scale-set-controller
├── kustomization.yaml               # Kustomize entry point
└── secret-github-app.sops.yaml      # SOPS-encrypted GitHub App credentials

k8s/configs/arc-runners/
├── namespace.yaml                    # arc-runners namespace for runner pods
├── helmrelease-runner-set.yaml      # HelmRelease for gha-runner-scale-set
├── kustomization.yaml               # Kustomize entry point
└── secret-github-app.sops.yaml      # SOPS-encrypted GitHub App credentials (runner namespace)
```

**Structure Decision**: Following existing homelab patterns, ARC controller goes in `k8s/infrastructure/` (cluster infrastructure) while runner scale sets go in `k8s/configs/` (workload configuration). This separation allows independent lifecycle management.

## Constitution Check (Post-Design)

*Re-check after Phase 1 design completion.*

| Principle | Status | Evidence |
|-----------|--------|----------|
| I. GitOps-First | ✅ PASS | HelmRelease manifests in contracts/, no imperative commands |
| II. Declarative Configuration | ✅ PASS | All resources defined as YAML, Kustomize structure maintained |
| III. Secrets Management | ✅ PASS | Secret template shows SOPS pattern, quickstart includes encryption steps |
| IV. Immutable Infrastructure | ✅ PASS | No Talos config changes required |
| V. Documentation as Code | ✅ PASS | research.md, data-model.md, quickstart.md created |

**Post-Design Gate Result**: PASS - Ready for task breakdown.

## Complexity Tracking

No violations to justify - implementation follows established patterns.

## Generated Artifacts

| Artifact | Path | Description |
|----------|------|-------------|
| Research | `specs/016-deploy-arc/research.md` | Technical decisions and alternatives |
| Data Model | `specs/016-deploy-arc/data-model.md` | Kubernetes resource definitions |
| Quickstart | `specs/016-deploy-arc/quickstart.md` | Deployment guide |
| Contracts | `specs/016-deploy-arc/contracts/` | Reference HelmRelease manifests |
