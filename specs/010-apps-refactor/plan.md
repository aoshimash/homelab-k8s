# Implementation Plan: Apps Directory Refactor

**Branch**: `010-apps-refactor` | **Date**: 2026-01-04 | **Spec**: [spec.md](./spec.md)
**Input**: Feature specification from `/specs/010-apps-refactor/spec.md`

## Summary

Refactor `k8s/apps/` directory structure to consolidate all resources deployed to the `audiobookshelf` namespace under a unified directory hierarchy. Move `k8s/apps/radigo/` components as sibling directories under `k8s/apps/audiobookshelf/` and reorganize core audiobookshelf resources into an `app/` subdirectory.

## Technical Context

**Language/Version**: YAML (Kubernetes manifests, Kustomize)
**Primary Dependencies**: Kustomize, Flux CD, SOPS
**Storage**: N/A (manifest refactoring only)
**Testing**: `kustomize build`, `flux reconcile`, manual verification
**Target Platform**: Kubernetes cluster with Flux CD GitOps
**Project Type**: Infrastructure as Code (GitOps manifests)
**Performance Goals**: N/A (no runtime impact)
**Constraints**: Zero downtime, preserve existing functionality
**Scale/Scope**: ~30 YAML files across 4 subdirectories

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-check after Phase 1 design.*

| Principle | Status | Notes |
|-----------|--------|-------|
| I. GitOps-First | ✅ PASS | All changes via Git commits, Flux reconciliation |
| II. Declarative Configuration | ✅ PASS | Using Kustomize for manifest management |
| III. Secrets Management | ✅ PASS | SOPS encrypted secrets preserved, path patterns still match `k8s/.*\.sops\.yaml$` |
| IV. Immutable Infrastructure | ✅ PASS | No node changes, manifest-only refactor |
| V. Documentation as Code | ✅ PASS | Spec and plan documented in `specs/` |

**Gate Result**: PASS - No violations detected.

## Project Structure

### Documentation (this feature)

```text
specs/010-apps-refactor/
├── plan.md              # This file
├── research.md          # Phase 0 output
├── data-model.md        # Phase 1 output
├── quickstart.md        # Phase 1 output
├── contracts/           # Phase 1 output
│   └── reconciliation.md
└── tasks.md             # Phase 2 output (created by /speckit.tasks)
```

### Source Code (repository root)

**Current Structure (Before)**:
```text
k8s/apps/
├── kustomization.yaml           # References audiobookshelf/ and radigo/
├── audiobookshelf/
│   ├── kustomization.yaml
│   ├── namespace.yaml
│   ├── deployment.yaml
│   ├── service.yaml
│   ├── ingress.yaml
│   └── pvc.yaml
└── radigo/
    ├── kustomization.yaml       # References shared-secrets, recording, metadata
    ├── shared-secrets/
    │   ├── kustomization.yaml
    │   ├── secret-ghcr.sops.yaml
    │   └── secret-audiobookshelf-api.sops.yaml
    ├── recording/
    │   ├── base/
    │   │   ├── kustomization.yaml
    │   │   └── cronjob.yaml
    │   ├── configmap-record-script.yaml
    │   ├── kustomization.yaml
    │   └── programs/
    │       ├── audrey/
    │       └── ijuin/
    └── metadata/
        ├── base/
        │   ├── kustomization.yaml
        │   └── job.yaml
        ├── configmap-metadata-script.yaml
        ├── kustomization.yaml
        └── programs/
            ├── audrey/
            └── ijuin/
```

**Target Structure (After)**:
```text
k8s/apps/
├── kustomization.yaml           # References only audiobookshelf/
└── audiobookshelf/
    ├── kustomization.yaml       # References namespace, app, shared-secrets, radigo-recorder, metadata-updater
    ├── namespace.yaml
    ├── app/                     # Core audiobookshelf application
    │   ├── kustomization.yaml
    │   ├── deployment.yaml
    │   ├── service.yaml
    │   ├── ingress.yaml
    │   └── pvc.yaml
    ├── shared-secrets/          # Moved from radigo/
    │   ├── kustomization.yaml
    │   ├── secret-ghcr.sops.yaml
    │   └── secret-audiobookshelf-api.sops.yaml
    ├── radigo-recorder/         # Renamed from radigo/recording/
    │   ├── base/
    │   │   ├── kustomization.yaml
    │   │   └── cronjob.yaml
    │   ├── configmap-record-script.yaml
    │   ├── kustomization.yaml
    │   └── programs/
    │       ├── audrey/
    │       └── ijuin/
    └── metadata-updater/        # Renamed from radigo/metadata/
        ├── base/
        │   ├── kustomization.yaml
        │   └── job.yaml
        ├── configmap-metadata-script.yaml
        ├── kustomization.yaml
        └── programs/
            ├── audrey/
            └── ijuin/
```

**Structure Decision**: Flat sibling directories under `audiobookshelf/` for clear separation of concerns without unnecessary nesting.

## External References to Update

| File | Current Path | New Path |
|------|--------------|----------|
| `.github/workflows/k8s-lint-security.yaml` | `apps/radigo/recording/programs` | `apps/audiobookshelf/radigo-recorder/programs` |
| `.github/workflows/k8s-lint-security.yaml` | `apps/radigo/metadata/programs` | `apps/audiobookshelf/metadata-updater/programs` |

## Complexity Tracking

> No Constitution violations requiring justification.

| Violation | Why Needed | Simpler Alternative Rejected Because |
|-----------|------------|-------------------------------------|
| N/A | N/A | N/A |
