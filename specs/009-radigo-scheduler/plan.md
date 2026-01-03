# Implementation Plan: Radigo Scheduled Recorder

**Branch**: `009-radigo-scheduler` | **Date**: 2026-01-04 | **Spec**: [spec.md](./spec.md)
**Input**: Feature specification from `/specs/009-radigo-scheduler/spec.md`

## Summary

Implement automated radiko program recording using Kubernetes CronJobs with radigo, saving recordings to audiobookshelf's podcasts PVC and triggering library scans via API. The system uses radiko's timefree feature, fetching program end times from radiko's schedule API and validating program titles before recording to handle hiatus/special broadcasts.

## Technical Context

**Language/Version**: Shell script (bash) for orchestration, radigo (Go binary) for recording  
**Primary Dependencies**: radigo Docker image (`yyoshiki41/radigo`), curl, jq for API calls  
**Storage**: Longhorn PVC (`audiobookshelf-podcasts`, 50Gi, ReadWriteOnce)  
**Testing**: Manual verification via Audiobookshelf UI and RSS feed  
**Target Platform**: Kubernetes (Talos Linux cluster)  
**Project Type**: Kubernetes manifests (CronJob, ConfigMap, Secret)  
**Performance Goals**: Up to 5 simultaneous recordings without resource conflicts  
**Constraints**: Must share PVC with audiobookshelf pod (same node scheduling required)  
**Scale/Scope**: Initial deployment with 2-3 programs, expandable to 5+ concurrent schedules

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-check after Phase 1 design.*

| Principle | Status | Notes |
|-----------|--------|-------|
| I. GitOps-First | ✅ PASS | All manifests in `k8s/apps/radigo/`, deployed via Flux |
| II. Declarative Configuration | ✅ PASS | CronJob + ConfigMap for schedules, no imperative commands |
| III. Secrets Management | ✅ PASS | Audiobookshelf API key in SOPS-encrypted Secret |
| IV. Immutable Infrastructure | ✅ PASS | Using existing radigo container image, no runtime modifications |
| V. Documentation as Code | ✅ PASS | Documentation in `docs/radigo.md` |

## Project Structure

### Documentation (this feature)

```text
specs/009-radigo-scheduler/
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
k8s/apps/radigo/
├── namespace.yaml                    # Namespace: radigo
├── configmap-schedules.yaml          # Recording schedules configuration
├── secret-audiobookshelf-api.sops.yaml  # Audiobookshelf API key (encrypted)
├── cronjob-arco.yaml                 # CronJob for オードリーのANN
├── cronjob-ijuin.yaml                # CronJob for 伊集院光深夜の馬鹿力
├── scripts/
│   └── record.sh                     # Recording orchestration script (in ConfigMap)
└── kustomization.yaml                # Kustomize configuration

docs/
└── radigo.md                         # Operational documentation
```

**Structure Decision**: Following existing `k8s/apps/` pattern with raw Kubernetes manifests. Each program gets its own CronJob for independent scheduling and failure isolation.

## Complexity Tracking

No constitution violations. Design follows existing cluster patterns.
