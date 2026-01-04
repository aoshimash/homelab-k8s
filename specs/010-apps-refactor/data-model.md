# Data Model: Apps Directory Refactor

**Feature**: 010-apps-refactor
**Date**: 2026-01-04

## Overview

This refactoring does not introduce new data models or entities. It reorganizes existing Kubernetes manifest files into a cleaner directory structure.

## Directory Entity Relationships

### Before Refactoring

```
k8s/apps/
├── kustomization.yaml
│   └── references: [audiobookshelf/, radigo/]
│
├── audiobookshelf/
│   └── kustomization.yaml
│       └── references: [namespace.yaml, pvc.yaml, deployment.yaml, service.yaml, ingress.yaml]
│
└── radigo/
    └── kustomization.yaml
        └── references: [shared-secrets/, recording/, metadata/]
```

### After Refactoring

```
k8s/apps/
├── kustomization.yaml
│   └── references: [audiobookshelf/]
│
└── audiobookshelf/
    └── kustomization.yaml
        └── references: [namespace.yaml, app/, shared-secrets/, radigo-recorder/, metadata-updater/]
```

## Kustomization Reference Map

### Root: `k8s/apps/kustomization.yaml`

| Before | After |
|--------|-------|
| `audiobookshelf/` | `audiobookshelf/` |
| `radigo/` | (removed) |

### Audiobookshelf: `k8s/apps/audiobookshelf/kustomization.yaml`

| Before | After |
|--------|-------|
| `namespace.yaml` | `namespace.yaml` |
| `pvc.yaml` | (moved to app/) |
| `deployment.yaml` | (moved to app/) |
| `service.yaml` | (moved to app/) |
| `ingress.yaml` | (moved to app/) |
| - | `app/` |
| - | `shared-secrets/` |
| - | `radigo-recorder/` |
| - | `metadata-updater/` |

### New: `k8s/apps/audiobookshelf/app/kustomization.yaml`

| Resources |
|-----------|
| `deployment.yaml` |
| `service.yaml` |
| `ingress.yaml` |
| `pvc.yaml` |

## File Migration Map

| Source | Destination |
|--------|-------------|
| `k8s/apps/audiobookshelf/deployment.yaml` | `k8s/apps/audiobookshelf/app/deployment.yaml` |
| `k8s/apps/audiobookshelf/service.yaml` | `k8s/apps/audiobookshelf/app/service.yaml` |
| `k8s/apps/audiobookshelf/ingress.yaml` | `k8s/apps/audiobookshelf/app/ingress.yaml` |
| `k8s/apps/audiobookshelf/pvc.yaml` | `k8s/apps/audiobookshelf/app/pvc.yaml` |
| `k8s/apps/radigo/shared-secrets/*` | `k8s/apps/audiobookshelf/shared-secrets/*` |
| `k8s/apps/radigo/recording/*` | `k8s/apps/audiobookshelf/radigo-recorder/*` |
| `k8s/apps/radigo/metadata/*` | `k8s/apps/audiobookshelf/metadata-updater/*` |

## Unchanged Resources

The following files are moved but their content remains unchanged:
- All `*.sops.yaml` secrets (SOPS encryption preserved)
- All ConfigMaps
- All CronJobs and Jobs
- All program-specific overlays

## Kustomization Files Requiring Updates

| File | Change Type |
|------|-------------|
| `k8s/apps/kustomization.yaml` | Modify (remove radigo/) |
| `k8s/apps/audiobookshelf/kustomization.yaml` | Modify (restructure resources) |
| `k8s/apps/audiobookshelf/app/kustomization.yaml` | Create new |
