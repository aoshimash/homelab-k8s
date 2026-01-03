# Implementation Plan: Audiobookshelf Podcast Server

**Branch**: `005-audiobookshelf-podcast` | **Date**: 2026-01-03 | **Spec**: [spec.md](./spec.md)
**Input**: Feature specification from `/specs/005-audiobookshelf-podcast/spec.md`

## Summary

Deploy Audiobookshelf as a self-hosted podcast server on the Kubernetes cluster using raw Kubernetes manifests managed by Flux CD. The service will be exposed via Tailscale Ingress using the existing ProxyGroup (`ingress-proxies`) for secure tailnet-only access. Persistent storage will be provisioned via Longhorn for podcast audio files (50Gi) and configuration/metadata.

## Technical Context

**Language/Version**: YAML (Kubernetes manifests)
**Primary Dependencies**: Flux CD, Kustomize, Tailscale Operator, Longhorn
**Storage**: Longhorn PersistentVolumeClaims (50Gi for podcasts, 1Gi for config/metadata)
**Testing**: Manual verification via kubectl and Tailscale access
**Target Platform**: Kubernetes (Talos Linux cluster)
**Project Type**: GitOps Kubernetes deployment
**Performance Goals**: Web UI accessible within 5 minutes of reconciliation, streaming starts within 5 seconds
**Constraints**: Tailnet-only access, SOPS encryption for any secrets
**Scale/Scope**: Single replica deployment, single-node cluster

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-check after Phase 1 design.*

| Principle | Status | Notes |
|-----------|--------|-------|
| I. GitOps-First | ✅ PASS | All manifests committed to Git, Flux reconciles |
| II. Declarative Configuration | ✅ PASS | Raw Kubernetes YAML with Kustomize |
| III. Secrets Management | ✅ PASS | No application secrets required (user accounts managed in-app) |
| IV. Immutable Infrastructure | ✅ PASS | No node-level changes required |
| V. Documentation as Code | ✅ PASS | Operational docs in `docs/audiobookshelf.md` |

**Gate Result**: PASS - No violations. Proceed to Phase 0.

## Project Structure

### Documentation (this feature)

```text
specs/005-audiobookshelf-podcast/
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
├── apps/
│   └── audiobookshelf/
│       ├── kustomization.yaml
│       ├── namespace.yaml
│       ├── deployment.yaml
│       ├── service.yaml
│       ├── ingress.yaml
│       └── pvc.yaml
└── flux/
    └── apps-kustomization.yaml  # New Flux Kustomization for apps

docs/
└── audiobookshelf.md            # Operational documentation
```

**Structure Decision**: Audiobookshelf is a user-facing application (not cluster infrastructure), so it belongs in `k8s/apps/` directory. This follows the constitution's directory structure pattern where `k8s/infrastructure/` is for cluster infrastructure and `k8s/apps/` is for user applications. A new Flux Kustomization will be created to manage the apps directory.

## Complexity Tracking

No constitution violations requiring justification.
