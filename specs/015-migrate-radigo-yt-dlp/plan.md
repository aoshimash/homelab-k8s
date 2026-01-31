# Implementation Plan: Migrate radigo to yt-dlp-rajiko

**Branch**: `015-migrate-radigo-yt-dlp` | **Date**: 2026-01-31 | **Spec**: [spec.md](./spec.md)  
**Input**: Feature specification from `/specs/015-migrate-radigo-yt-dlp/spec.md`

## Summary

Migrate the radiko recording system from the broken radigo tool (v0.12.0) to yt-dlp with yt-dlp-rajiko plugin. The migration involves creating a new Docker image, updating the recording script to use yt-dlp CLI, and updating Kubernetes CronJob configurations while maintaining compatibility with existing storage and scheduling infrastructure.

## Technical Context

**Language/Version**: Shell script (bash), Dockerfile  
**Primary Dependencies**: yt-dlp (2025.02.19+), yt-dlp-rajiko plugin (v1.11), ffmpeg  
**Storage**: Existing PersistentVolumeClaim (audiobookshelf-podcasts)  
**Testing**: Manual verification via kubectl exec and CronJob trigger  
**Target Platform**: Kubernetes (homelab cluster on Talos Linux)  
**Project Type**: Infrastructure/GitOps configuration  
**Performance Goals**: Complete recording within radiko timefree window (7 days)  
**Constraints**: Must work without premium radiko account (area-free mode)  
**Scale/Scope**: 4 radio programs (ijuin, audrey, arco, ariyoshi)

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-check after Phase 1 design.*

| Principle | Status | Notes |
|-----------|--------|-------|
| I. GitOps-First | ✅ PASS | All changes via Git commits, Flux reconciliation |
| II. Declarative Configuration | ✅ PASS | Kustomize overlays, ConfigMap for scripts |
| III. Secrets Management | ✅ PASS | No new secrets required (area-free mode) |
| IV. Immutable Infrastructure | ✅ PASS | New Docker image, no runtime modifications |
| V. Documentation as Code | ✅ PASS | Spec and plan documentation in specs/ |

**Gate Result**: PASSED - Proceed to Phase 0

### Post-Phase 1 Re-check

| Principle | Status | Verification |
|-----------|--------|--------------|
| I. GitOps-First | ✅ PASS | All artifacts in Git, Flux reconciliation |
| II. Declarative Configuration | ✅ PASS | Kustomize structure preserved |
| III. Secrets Management | ✅ PASS | No secrets needed (area-free) |
| IV. Immutable Infrastructure | ✅ PASS | New Docker image, no runtime changes |
| V. Documentation as Code | ✅ PASS | research.md, data-model.md, quickstart.md created |

**Final Gate Result**: PASSED - Ready for Phase 2 (/speckit.tasks)

## Project Structure

### Documentation (this feature)

```text
specs/015-migrate-radigo-yt-dlp/
├── spec.md              # Feature specification
├── plan.md              # This file
├── research.md          # Phase 0 output
├── data-model.md        # Phase 1 output
├── quickstart.md        # Phase 1 output
├── checklists/          # Quality checklists
│   └── requirements.md
└── tasks.md             # Phase 2 output (created by /speckit.tasks)
```

### Source Code (repository root)

```text
images/
└── yt-dlp-rajiko/
    └── Dockerfile                    # NEW: yt-dlp + rajiko plugin image

k8s/apps/audiobookshelf/radigo-recorder/
├── base/
│   ├── cronjob.yaml                  # UPDATE: image reference, backoffLimit
│   └── kustomization.yaml
├── configmap-record-script.yaml      # UPDATE: yt-dlp based recording script
├── kustomization.yaml
└── programs/
    ├── arco/
    ├── audrey/
    └── ijuin/

images/radigo/                         # DELETE: old radigo image
└── Dockerfile
```

**Structure Decision**: Kubernetes GitOps pattern - Docker image in `images/`, Kustomize manifests in `k8s/apps/`. Replace existing radigo components in-place.

## Complexity Tracking

No violations - standard GitOps pattern with straightforward component replacement.
