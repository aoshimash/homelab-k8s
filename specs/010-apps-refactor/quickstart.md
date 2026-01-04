# Quickstart: Apps Directory Refactor

**Feature**: 010-apps-refactor
**Date**: 2026-01-04

## Overview

This guide provides step-by-step instructions to implement the apps directory refactoring.

## Prerequisites

- Git repository cloned locally
- On feature branch `010-apps-refactor`
- Flux CD access to the cluster (for verification)

## Implementation Steps

### Step 1: Create New Directory Structure

```bash
# Create app/ subdirectory for audiobookshelf core
mkdir -p k8s/apps/audiobookshelf/app

# Create directories for radigo components (moved from radigo/)
mkdir -p k8s/apps/audiobookshelf/shared-secrets
mkdir -p k8s/apps/audiobookshelf/radigo-recorder/base
mkdir -p k8s/apps/audiobookshelf/radigo-recorder/programs/audrey
mkdir -p k8s/apps/audiobookshelf/radigo-recorder/programs/ijuin
mkdir -p k8s/apps/audiobookshelf/metadata-updater/base
mkdir -p k8s/apps/audiobookshelf/metadata-updater/programs/audrey
mkdir -p k8s/apps/audiobookshelf/metadata-updater/programs/ijuin
```

### Step 2: Move Audiobookshelf Core Files

```bash
# Move core app files to app/ subdirectory
git mv k8s/apps/audiobookshelf/deployment.yaml k8s/apps/audiobookshelf/app/
git mv k8s/apps/audiobookshelf/service.yaml k8s/apps/audiobookshelf/app/
git mv k8s/apps/audiobookshelf/ingress.yaml k8s/apps/audiobookshelf/app/
git mv k8s/apps/audiobookshelf/pvc.yaml k8s/apps/audiobookshelf/app/
```

### Step 3: Move Radigo Components

```bash
# Move shared-secrets
git mv k8s/apps/radigo/shared-secrets/* k8s/apps/audiobookshelf/shared-secrets/

# Move recording to radigo-recorder
git mv k8s/apps/radigo/recording/base/* k8s/apps/audiobookshelf/radigo-recorder/base/
git mv k8s/apps/radigo/recording/configmap-record-script.yaml k8s/apps/audiobookshelf/radigo-recorder/
git mv k8s/apps/radigo/recording/kustomization.yaml k8s/apps/audiobookshelf/radigo-recorder/
git mv k8s/apps/radigo/recording/programs/audrey/* k8s/apps/audiobookshelf/radigo-recorder/programs/audrey/
git mv k8s/apps/radigo/recording/programs/ijuin/* k8s/apps/audiobookshelf/radigo-recorder/programs/ijuin/

# Move metadata to metadata-updater
git mv k8s/apps/radigo/metadata/base/* k8s/apps/audiobookshelf/metadata-updater/base/
git mv k8s/apps/radigo/metadata/configmap-metadata-script.yaml k8s/apps/audiobookshelf/metadata-updater/
git mv k8s/apps/radigo/metadata/kustomization.yaml k8s/apps/audiobookshelf/metadata-updater/
git mv k8s/apps/radigo/metadata/programs/audrey/* k8s/apps/audiobookshelf/metadata-updater/programs/audrey/
git mv k8s/apps/radigo/metadata/programs/ijuin/* k8s/apps/audiobookshelf/metadata-updater/programs/ijuin/
```

### Step 4: Remove Old Radigo Directory

```bash
# Remove empty radigo directory
rm -rf k8s/apps/radigo/
```

### Step 5: Update Kustomization Files

**k8s/apps/kustomization.yaml**:
```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - audiobookshelf/
```

**k8s/apps/audiobookshelf/kustomization.yaml**:
```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - namespace.yaml
  - app/
  - shared-secrets/
  - radigo-recorder/
  - metadata-updater/
```

**k8s/apps/audiobookshelf/app/kustomization.yaml** (new file):
```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - deployment.yaml
  - service.yaml
  - ingress.yaml
  - pvc.yaml
```

### Step 6: Update CI/CD Workflow

**`.github/workflows/k8s-lint-security.yaml`**:

Change line 53:
```yaml
# Before
skip-dirs: "apps/radigo/recording/programs,apps/radigo/metadata/programs"

# After
skip-dirs: "apps/audiobookshelf/radigo-recorder/programs,apps/audiobookshelf/metadata-updater/programs"
```

### Step 7: Verify Kustomize Build

```bash
# Test kustomize build locally
kustomize build k8s/apps/

# Should output all resources without errors
```

### Step 8: Commit and Push

```bash
git add -A
git commit -m "refactor(apps): consolidate audiobookshelf namespace resources

- Move audiobookshelf core to app/ subdirectory
- Move radigo/recording to radigo-recorder/
- Move radigo/metadata to metadata-updater/
- Move radigo/shared-secrets to audiobookshelf level
- Update all kustomization.yaml references
- Update CI skip-dirs for new paths"

git push origin 010-apps-refactor
```

### Step 9: Verify Flux Reconciliation

```bash
# Force Flux to reconcile
flux reconcile kustomization apps --with-source

# Check status
flux get kustomizations

# Verify resources
kubectl get all -n audiobookshelf
```

## Verification Checklist

- [ ] `kustomize build k8s/apps/` succeeds
- [ ] CI lint and security checks pass
- [ ] Flux reconciliation completes without errors
- [ ] Audiobookshelf deployment is running
- [ ] CronJobs are scheduled correctly
- [ ] Secrets are properly decrypted (not showing SOPS metadata)

## Rollback

If issues occur, revert the commit:

```bash
git revert HEAD
git push origin 010-apps-refactor
flux reconcile kustomization apps --with-source
```
