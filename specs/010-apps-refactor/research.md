# Research: Apps Directory Refactor

**Feature**: 010-apps-refactor
**Date**: 2026-01-04

## Research Summary

This refactoring is a straightforward directory restructuring with no technical unknowns. All technologies (Kustomize, Flux, SOPS) are already in use in the repository.

## Decisions

### 1. Directory Move Strategy

**Decision**: Use `git mv` for all file moves to preserve Git history.

**Rationale**: Git tracks file moves better when using `git mv` command. This preserves blame history and makes it easier to trace changes across the refactoring.

**Alternatives Considered**:
- Manual copy and delete: Would lose Git history for moved files
- Single commit with all changes: Chosen approach for atomic change

### 2. Kustomization Reference Updates

**Decision**: Update all `kustomization.yaml` files to use relative paths from their new locations.

**Rationale**: Kustomize uses relative paths from the kustomization.yaml file location. All paths remain relative, no absolute paths needed.

**Files Requiring Updates**:
- `k8s/apps/kustomization.yaml` - Remove `radigo/`, keep `audiobookshelf/`
- `k8s/apps/audiobookshelf/kustomization.yaml` - Add app/, shared-secrets/, radigo-recorder/, metadata-updater/
- `k8s/apps/audiobookshelf/app/kustomization.yaml` - New file referencing moved resources

### 3. SOPS Path Compatibility

**Decision**: No changes needed to `.sops.yaml` configuration.

**Rationale**: The SOPS configuration uses regex pattern `k8s/.*\.sops\.yaml$` which matches any `.sops.yaml` file under `k8s/` directory. The new paths `k8s/apps/audiobookshelf/shared-secrets/*.sops.yaml` still match this pattern.

**Verification**: The regex `k8s/.*\.sops\.yaml$` matches:
- Current: `k8s/apps/radigo/shared-secrets/secret-ghcr.sops.yaml` ✅
- New: `k8s/apps/audiobookshelf/shared-secrets/secret-ghcr.sops.yaml` ✅

### 4. CI/CD Updates

**Decision**: Update Trivy skip-dirs in `.github/workflows/k8s-lint-security.yaml`.

**Rationale**: The Trivy security scan skips certain directories containing program-specific overlays. These paths must be updated to match the new directory structure.

**Changes Required**:
```yaml
# Before
skip-dirs: "apps/radigo/recording/programs,apps/radigo/metadata/programs"

# After
skip-dirs: "apps/audiobookshelf/radigo-recorder/programs,apps/audiobookshelf/metadata-updater/programs"
```

### 5. Namespace Handling

**Decision**: Keep `namespace.yaml` at `audiobookshelf/` root level, not inside `app/`.

**Rationale**: The namespace resource is shared by all components (app, radigo-recorder, metadata-updater, shared-secrets). Placing it at the audiobookshelf root makes this relationship clear.

**Alternatives Considered**:
- Put namespace.yaml in app/: Would suggest it belongs only to the app component
- Reference namespace.yaml from multiple places: Unnecessary duplication

## No Unknowns Remaining

All technical decisions have been made. No NEEDS CLARIFICATION items remain.
