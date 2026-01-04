# Feature Specification: Apps Directory Refactor

**Feature Branch**: `010-apps-refactor`  
**Created**: 2026-01-04  
**Status**: Draft  
**Input**: User description: "appsディレクトリ以下をリファクタ - 現在audiobookshelfもradigo/recordingもradigo/metadata もaudiobookshelf namespace以下にデプロイしているので、すべてを同じフォルダの下に入れてその中のサブフォルダで分けたい"

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Unified Directory Structure for Audiobookshelf Namespace (Priority: P1)

As a Kubernetes infrastructure maintainer, I want all resources deployed to the `audiobookshelf` namespace to be organized under a single directory structure, so that I can easily understand and manage related resources together.

**Why this priority**: This is the core requirement - consolidating resources that share the same namespace into a logical directory hierarchy improves maintainability and reduces confusion about resource relationships.

**Independent Test**: Can be verified by examining the new directory structure and confirming that all resources under `k8s/apps/audiobookshelf/` are deployed to the `audiobookshelf` namespace, while maintaining the same Kubernetes resources and functionality.

**Acceptance Scenarios**:

1. **Given** the current directory structure with `k8s/apps/audiobookshelf/` and `k8s/apps/radigo/`, **When** the refactoring is complete, **Then** all audiobookshelf namespace resources are under `k8s/apps/audiobookshelf/` with clear subdirectories.

2. **Given** the refactored structure, **When** Flux reconciles the Kustomization, **Then** all resources are correctly deployed to the `audiobookshelf` namespace without errors.

3. **Given** the refactored structure, **When** a developer looks for radigo recording configuration, **Then** they can find it at `k8s/apps/audiobookshelf/radigo-recorder/`.

---

### User Story 2 - Preserved Functionality After Refactor (Priority: P1)

As a Kubernetes infrastructure maintainer, I want all existing functionality to work exactly as before after the refactoring, so that the structure change doesn't break running services.

**Why this priority**: Equally critical - a refactor that breaks functionality defeats its purpose.

**Independent Test**: Can be verified by deploying the refactored manifests and confirming that audiobookshelf deployment, radigo recording CronJobs, and radigo metadata Jobs continue to function correctly.

**Acceptance Scenarios**:

1. **Given** the refactored directory structure, **When** Flux reconciles, **Then** the audiobookshelf deployment continues running without changes.

2. **Given** the refactored directory structure, **When** a radigo recording CronJob triggers, **Then** the recording completes successfully and files are uploaded to audiobookshelf.

3. **Given** the refactored directory structure, **When** a radigo metadata Job runs, **Then** the episode metadata is correctly updated in audiobookshelf.

---

### User Story 3 - Clear Separation of Concerns (Priority: P2)

As a Kubernetes infrastructure maintainer, I want subdirectories to clearly separate different components (audiobookshelf app, radigo recording, radigo metadata, shared secrets), so that I can work on individual components without confusion.

**Why this priority**: Important for long-term maintainability, but secondary to the core structural change.

**Independent Test**: Can be verified by examining the directory structure and confirming each component has its own isolated subdirectory with related resources only.

**Acceptance Scenarios**:

1. **Given** the refactored structure, **When** I navigate to `k8s/apps/audiobookshelf/app/`, **Then** I find only the core audiobookshelf deployment resources (deployment, service, ingress, pvc).

2. **Given** the refactored structure, **When** I navigate to `k8s/apps/audiobookshelf/radigo-recorder/`, **Then** I find only recording-related resources (CronJobs, ConfigMaps, program overlays).

3. **Given** the refactored structure, **When** I navigate to `k8s/apps/audiobookshelf/metadata-updater/`, **Then** I find only metadata update resources (Jobs, ConfigMaps, program overlays).

---

### Edge Cases

- What happens if kustomization.yaml references have broken paths after the move?
  - All path references must be updated to reflect new directory structure
- What happens if there are external references to the old paths (e.g., in CI/CD)?
  - Check and update any references in `.github/workflows/` or other configuration files
- What happens if SOPS encrypted secrets need path updates?
  - `.sops.yaml` creation_rules should be checked for path patterns

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: System MUST reorganize `k8s/apps/radigo/` contents under `k8s/apps/audiobookshelf/` as sibling directories (recording, metadata, shared-secrets)
- **FR-002**: System MUST move core audiobookshelf resources to `k8s/apps/audiobookshelf/app/` subdirectory
- **FR-003**: System MUST update all kustomization.yaml files to reflect new paths
- **FR-004**: System MUST preserve the namespace definition at the audiobookshelf level
- **FR-005**: System MUST maintain shared-secrets accessibility from both recording and metadata components
- **FR-006**: System MUST update the root `k8s/apps/kustomization.yaml` to reference only `audiobookshelf/`
- **FR-007**: System MUST verify all SOPS encrypted files remain decryptable after restructuring

### Key Entities

- **Audiobookshelf App**: Core audiobookshelf deployment, service, ingress, and PVC
- **Radigo Recording**: CronJobs for recording radiko programs with per-program overlays
- **Radigo Metadata**: Jobs for updating audiobookshelf episode metadata with per-program overlays
- **Shared Secrets**: GHCR image pull secret and Audiobookshelf API credentials used by radigo components

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: All resources previously deployed to `audiobookshelf` namespace are still deployed correctly after refactoring
- **SC-002**: Directory structure follows the pattern `k8s/apps/audiobookshelf/{app,radigo-recorder,metadata-updater,shared-secrets}/`
- **SC-003**: `flux reconcile` completes without errors after applying the new structure
- **SC-004**: No duplicate resources exist after refactoring
- **SC-005**: All CronJobs and Jobs execute successfully after the structural change

## Proposed Directory Structure

```
k8s/apps/
└── audiobookshelf/
    ├── kustomization.yaml       # Root kustomization including all subdirectories
    ├── namespace.yaml           # Namespace definition
    ├── app/                     # Core audiobookshelf application
    │   ├── kustomization.yaml
    │   ├── deployment.yaml
    │   ├── service.yaml
    │   ├── ingress.yaml
    │   └── pvc.yaml
    ├── shared-secrets/          # Shared secrets for recording/metadata
    │   ├── kustomization.yaml
    │   ├── secret-ghcr.sops.yaml
    │   └── secret-audiobookshelf-api.sops.yaml
    ├── radigo-recorder/         # Radiko recording CronJobs
    │   ├── base/
    │   │   ├── kustomization.yaml
    │   │   └── cronjob.yaml
    │   ├── configmap-record-script.yaml
    │   ├── kustomization.yaml
    │   └── programs/
    │       ├── audrey/
    │       │   ├── kustomization.yaml
    │       │   └── program.yaml
    │       └── ijuin/
    │           ├── kustomization.yaml
    │           └── program.yaml
    └── metadata-updater/        # Audiobookshelf metadata update Jobs
        ├── base/
        │   ├── kustomization.yaml
        │   └── job.yaml
        ├── configmap-metadata-script.yaml
        ├── kustomization.yaml
        └── programs/
            ├── audrey/
            │   ├── kustomization.yaml
            │   └── job.yaml
            └── ijuin/
                ├── kustomization.yaml
                └── job.yaml
```

## Assumptions

- The `audiobookshelf` namespace is the only namespace used by all these resources
- Flux watches `k8s/apps/` directory for changes
- SOPS path patterns in `.sops.yaml` use `*.sops.yaml` glob patterns that don't depend on specific directory paths
- No external systems depend on the current directory structure
