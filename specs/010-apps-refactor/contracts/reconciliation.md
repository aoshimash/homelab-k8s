# Contract: Flux Reconciliation

**Feature**: 010-apps-refactor
**Date**: 2026-01-04

## Overview

This contract defines the expected behavior of Flux CD reconciliation after the directory refactoring is complete.

## Pre-conditions

1. All files have been moved to their new locations
2. All `kustomization.yaml` files have been updated with correct paths
3. CI/CD workflow has been updated with new skip-dirs paths
4. Changes have been committed and pushed to the repository

## Expected Behavior

### Flux Kustomization Reconciliation

When Flux reconciles the `apps` Kustomization:

1. **Source Detection**
   - Flux detects changes in `k8s/apps/` directory
   - GitRepository source is updated with new commit

2. **Kustomize Build**
   - `kustomize build k8s/apps/` succeeds without errors
   - All relative paths resolve correctly
   - No duplicate resource definitions

3. **Resource Application**
   - All resources are applied to `audiobookshelf` namespace
   - No resources are deleted (moved resources maintain same metadata)
   - No orphaned resources from old paths

### Resource Verification

| Resource Type | Expected Count | Namespace |
|---------------|----------------|-----------|
| Namespace | 1 | - |
| Deployment | 1 | audiobookshelf |
| Service | 1 | audiobookshelf |
| Ingress | 1 | audiobookshelf |
| PersistentVolumeClaim | 1 | audiobookshelf |
| Secret | 2 | audiobookshelf |
| ConfigMap | 2 | audiobookshelf |
| CronJob | 2 | audiobookshelf |
| Job | 0 (Jobs are created on-demand) | audiobookshelf |

## Post-conditions

1. `flux get kustomizations` shows `apps` as Ready
2. `kubectl get all -n audiobookshelf` shows all expected resources
3. No error events in `kubectl get events -n audiobookshelf`
4. Audiobookshelf pod is Running
5. CronJobs are scheduled with correct next run times

## Failure Scenarios

### Path Resolution Failure

**Symptom**: Flux shows `kustomize build failed` error
**Cause**: Incorrect relative path in kustomization.yaml
**Resolution**: Check and fix path references

### Duplicate Resource

**Symptom**: `resource already exists` error
**Cause**: Old and new paths both referenced
**Resolution**: Ensure only new paths are referenced in kustomization files

### Missing Secret Decryption

**Symptom**: Secret resources show encrypted values
**Cause**: SOPS decryption not configured for new path
**Resolution**: Verify `.sops.yaml` patterns match new paths (should be OK with `k8s/.*\.sops\.yaml$`)

## Verification Commands

```bash
# Check Flux reconciliation status
flux get kustomizations

# Force reconciliation
flux reconcile kustomization apps --with-source

# Verify resources
kubectl get all -n audiobookshelf

# Check for events/errors
kubectl get events -n audiobookshelf --sort-by='.lastTimestamp'

# Verify secrets are decrypted
kubectl get secret -n audiobookshelf -o yaml | grep -v 'sops:'
```
