# Longhorn - Storage Operations

This document describes Longhorn storage operations, troubleshooting, and upgrade procedures for the homelab-k8s repository.

## Overview

[Longhorn](https://longhorn.io/) is a distributed block storage system for Kubernetes, providing persistent storage for stateful workloads.

## Current Configuration

- **Version**: v1.7.3 (as of 2026-01-03)
- **Storage Class**: `longhorn`
- **Default Replica Count**: 1 (single-node cluster)
- **Backup Target**: Cloudflare R2 (S3-compatible)
- **Backup Schedule**: Daily at 02:00 UTC
- **Backup Retention**: 30 days

## Configuration Files

- **HelmRelease**: `k8s/infrastructure/longhorn/helmrelease.yaml`
- **Namespace**: `k8s/infrastructure/longhorn/namespace.yaml`
- **Credentials**: `k8s/infrastructure/longhorn/secret-r2-credentials.sops.yaml`

## Operations

### Check Longhorn Status

```bash
# Check all Longhorn pods
kubectl get pods -n longhorn-system

# Check Longhorn manager status
kubectl get pods -n longhorn-system -l app=longhorn-manager

# Check Longhorn UI
kubectl get pods -n longhorn-system -l app=longhorn-ui

# Check Longhorn version
kubectl get settings.longhorn.io -n longhorn-system current-longhorn-version -o jsonpath='{.value}'
```

### Access Longhorn UI

```bash
# Port forward to Longhorn UI
kubectl port-forward -n longhorn-system svc/longhorn-frontend 8080:80

# Access at http://localhost:8080
```

### Check Storage Volumes

```bash
# List all volumes
kubectl get volumes.longhorn.io -n longhorn-system

# Describe a specific volume
kubectl describe volume.longhorn.io <volume-name> -n longhorn-system

# Check PVC status
kubectl get pvc --all-namespaces
```

### Force Flux Reconciliation

```bash
# Reconcile Longhorn HelmRelease
flux reconcile helmrelease longhorn -n longhorn-system

# Reconcile infrastructure kustomization
flux reconcile kustomization infrastructure --with-source
```

## Troubleshooting

### Pods Not Starting

#### Driver-Deployer Stuck in Init:0/1

**Symptom**: `longhorn-driver-deployer` pod stuck in `Init:0/1` state

**Cause**: Init container waiting for `longhorn-backend` service to be ready, but `longhorn-manager` is not running.

**Diagnosis**:
```bash
# Check driver-deployer pod
kubectl describe pod -n longhorn-system -l app=longhorn-driver-deployer

# Check init container logs
kubectl logs -n longhorn-system <pod-name> -c wait-longhorn-manager

# Check if backend service exists
kubectl get svc -n longhorn-system longhorn-backend
```

**Solution**: Fix the underlying `longhorn-manager` issue first (see below).

#### Manager in CrashLoopBackOff

**Symptom**: `longhorn-manager` pod in `CrashLoopBackOff` state

**Common Causes**:

1. **Version Upgrade Incompatibility** (Most Common)
   - Error: `failed to upgrade since upgrading from vX.Y.Z to vA.B.C for minor version is not supported`
   - Longhorn requires sequential minor version upgrades

2. **Database Corruption**
   - Check logs for database-related errors

3. **Resource Constraints**
   - Check node resources: `kubectl top nodes`

**Diagnosis**:
```bash
# Check manager pod logs
kubectl logs -n longhorn-system -l app=longhorn-manager --tail=100

# Check pod events
kubectl describe pod -n longhorn-system -l app=longhorn-manager

# Check current version vs configured version
kubectl get settings.longhorn.io -n longhorn-system current-longhorn-version -o jsonpath='{.value}'
kubectl get helmrelease longhorn -n longhorn-system -o jsonpath='{.spec.chart.spec.version}'
```

**Solution for Version Upgrade Issues**:

1. **Identify current version**:
   ```bash
   kubectl get settings.longhorn.io -n longhorn-system current-longhorn-version -o jsonpath='{.value}'
   ```

2. **Revert HelmRelease to current version**:
   - Edit `k8s/infrastructure/longhorn/helmrelease.yaml`
   - Set `version` to match current cluster version
   - Commit and push changes
   - Flux will reconcile and restore stability

3. **Plan sequential upgrade** (see Upgrade Procedures below)

### Storage Issues

#### PVC Stuck in Pending

**Symptom**: PVC remains in `Pending` state

**Diagnosis**:
```bash
# Check PVC events
kubectl describe pvc <pvc-name> -n <namespace>

# Check StorageClass
kubectl get storageclass longhorn

# Check Longhorn volumes
kubectl get volumes.longhorn.io -n longhorn-system
```

**Common Issues**:
- Longhorn manager not running → Fix manager first
- StorageClass not found → Check HelmRelease reconciliation
- Node not ready → Check node status: `kubectl get nodes`

#### Volume Not Attaching

**Symptom**: Volume exists but pod cannot mount it

**Diagnosis**:
```bash
# Check volume attachment
kubectl get volumeattachments.longhorn.io -n longhorn-system

# Check instance manager
kubectl get pods -n longhorn-system -l longhorn.io/component=instance-manager

# Check CSI plugin
kubectl get pods -n longhorn-system -l app=longhorn-csi-plugin
```

### Backup Issues

#### Backup Target Not Accessible

**Symptom**: Backups failing with authentication errors

**Diagnosis**:
```bash
# Check backup target setting
kubectl get settings.longhorn.io -n longhorn-system backup-target -o jsonpath='{.value}'

# Check credentials secret
kubectl get secret -n longhorn-system longhorn-r2-credentials

# Check backup target status in Longhorn UI
```

**Solution**:
- Verify R2 credentials are correct
- Check SOPS decryption is working: `kubectl get secret -n longhorn-system longhorn-r2-credentials -o yaml`
- Ensure Flux SOPS decryption is configured in infrastructure kustomization

## Upgrade Procedures

### ⚠️ Important: Sequential Upgrades Required

Longhorn **does not support** direct upgrades across multiple minor versions. You must upgrade sequentially:

- v1.7.x → v1.8.x → v1.9.x → v1.10.x

### Pre-Upgrade Checklist

1. **Verify current version**:
   ```bash
   kubectl get settings.longhorn.io -n longhorn-system current-longhorn-version -o jsonpath='{.value}'
   ```

2. **Check all volumes are healthy**:
   ```bash
   kubectl get volumes.longhorn.io -n longhorn-system
   # All volumes should be in "attached" or "detached" state, not "error"
   ```

3. **Create backup** (if using R2 backups):
   - Use Longhorn UI to create manual backup
   - Or verify automatic backups are running

4. **Check Kubernetes version compatibility**:
   - Longhorn v1.10.x requires Kubernetes v1.21+
   - Current cluster: `kubectl get nodes -o jsonpath='{.items[0].status.nodeInfo.kubeletVersion}'`

### Upgrade Steps

#### Step 1: Upgrade to v1.8.x

1. **Update HelmRelease**:
   ```yaml
   # k8s/infrastructure/longhorn/helmrelease.yaml
   version: 1.8.4  # Use latest v1.8.x version
   ```

2. **Commit and push**:
   ```bash
   git add k8s/infrastructure/longhorn/helmrelease.yaml
   git commit -m "chore: upgrade Longhorn to v1.8.4"
   git push
   ```

3. **Monitor upgrade**:
   ```bash
   # Watch pods
   kubectl get pods -n longhorn-system -w

   # Check version
   kubectl get settings.longhorn.io -n longhorn-system current-longhorn-version -o jsonpath='{.value}'
   ```

4. **Verify health**:
   ```bash
   # All pods should be Running
   kubectl get pods -n longhorn-system

   # Volumes should be healthy
   kubectl get volumes.longhorn.io -n longhorn-system
   ```

5. **Wait for stability** (recommended: 24 hours)

#### Step 2: Upgrade to v1.9.x

Repeat Step 1 with v1.9.x version (e.g., `1.9.3`)

#### Step 3: Upgrade to v1.10.x

Repeat Step 1 with v1.10.x version (e.g., `1.10.1`)

### Post-Upgrade Verification

1. **Check all pods are running**:
   ```bash
   kubectl get pods -n longhorn-system
   # All pods should be Ready
   ```

2. **Verify volumes**:
   ```bash
   kubectl get volumes.longhorn.io -n longhorn-system
   # All volumes should be healthy
   ```

3. **Test PVC creation**:
   ```bash
   # Create test PVC
   kubectl apply -f - <<EOF
   apiVersion: v1
   kind: PersistentVolumeClaim
   metadata:
     name: test-pvc
     namespace: default
   spec:
     accessModes:
       - ReadWriteOnce
     storageClassName: longhorn
     resources:
       requests:
         storage: 1Gi
   EOF

   # Verify PVC is bound
   kubectl get pvc test-pvc
   ```

4. **Test backup/restore** (if configured):
   - Create test volume in Longhorn UI
   - Create backup
   - Verify backup appears in R2

### Rollback Procedure

If upgrade fails:

1. **Revert HelmRelease** to previous working version:
   ```bash
   # Edit helmrelease.yaml
   # Set version back to previous version
   git commit -m "revert: rollback Longhorn to vX.Y.Z"
   git push
   ```

2. **Force reconciliation**:
   ```bash
   flux reconcile helmrelease longhorn -n longhorn-system
   ```

3. **Monitor recovery**:
   ```bash
   kubectl get pods -n longhorn-system -w
   ```

## Maintenance

### Clean Up Old Engine Images

```bash
# List engine images
kubectl get engineimages.longhorn.io -n longhorn-system

# Delete unused engine images (be careful!)
kubectl delete engineimage.longhorn.io <image-name> -n longhorn-system
```

### Clean Up Orphaned Resources

```bash
# Check for orphaned replicas
kubectl get replicas.longhorn.io -n longhorn-system

# Check for orphaned engines
kubectl get engines.longhorn.io -n longhorn-system
```

## References

- [Longhorn Documentation](https://longhorn.io/docs/)
- [Longhorn Upgrade Guide](https://longhorn.io/docs/1.10.0/deploy/upgrade/)
- [Longhorn Troubleshooting](https://longhorn.io/docs/1.10.0/troubleshooting/)
