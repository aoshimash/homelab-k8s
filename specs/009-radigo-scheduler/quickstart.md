# Quickstart: Radigo Scheduled Recorder

## Prerequisites

- [x] Kubernetes cluster running (Talos Linux)
- [x] Flux CD installed and configured
- [x] Audiobookshelf deployed with `audiobookshelf-podcasts` PVC
- [x] SOPS age key configured in cluster

## Setup Steps

### 1. Get Audiobookshelf API Key and Library ID

1. Access Audiobookshelf web UI
2. Go to **Settings** → **Users** → Select your user → **API Keys**
3. Create new API key, copy the value
4. Go to **Libraries** → Select podcasts library → Note the Library ID from URL

### 2. Create SOPS-encrypted Secret

```bash
# Create secret file
cat > k8s/apps/radigo/secret-audiobookshelf-api.sops.yaml << 'EOF'
apiVersion: v1
kind: Secret
metadata:
  name: radigo-audiobookshelf-api
  namespace: radigo
type: Opaque
stringData:
  api_key: "YOUR_API_KEY_HERE"
  library_id: "YOUR_LIBRARY_ID_HERE"
  base_url: "http://audiobookshelf.audiobookshelf.svc.cluster.local"
EOF

# Encrypt with SOPS
sops -e -i k8s/apps/radigo/secret-audiobookshelf-api.sops.yaml
```

### 3. Create Program Directories

SSH into audiobookshelf pod or use kubectl exec:

```bash
kubectl exec -n audiobookshelf deploy/audiobookshelf -- mkdir -p /podcasts/arco /podcasts/ijuin
```

Or let the recording script create them automatically.

### 4. Deploy via GitOps

```bash
# Commit and push
git add k8s/apps/radigo/
git commit -m "feat: add radigo scheduled recorder"
git push

# Wait for Flux reconciliation
flux reconcile kustomization apps --with-source
```

### 5. Verify Deployment

```bash
# Check namespace
kubectl get ns radigo

# Check CronJobs
kubectl get cronjobs -n radigo

# Check secret (encrypted at rest)
kubectl get secret -n radigo

# Check ConfigMap
kubectl describe cm radigo-schedules -n radigo
```

## Manual Test Recording

To test without waiting for scheduled time:

```bash
# Create a manual job from CronJob
kubectl create job --from=cronjob/radigo-arco radigo-arco-test -n radigo

# Watch job progress
kubectl get jobs -n radigo -w

# Check logs
kubectl logs -n radigo -l job-name=radigo-arco-test -f
```

## Adding a New Program

1. Edit `k8s/apps/radigo/configmap-schedules.yaml`:
   ```yaml
   # Add new program entry
   ```

2. Create new CronJob file `k8s/apps/radigo/cronjob-newprogram.yaml`

3. Update `k8s/apps/radigo/kustomization.yaml` to include new CronJob

4. Commit and push - Flux handles the rest

## Troubleshooting

### Recording not appearing in Audiobookshelf

1. Check job completed successfully:
   ```bash
   kubectl get jobs -n radigo
   ```

2. Check logs for errors:
   ```bash
   kubectl logs -n radigo job/<job-name>
   ```

3. Verify file exists:
   ```bash
   kubectl exec -n audiobookshelf deploy/audiobookshelf -- ls -la /podcasts/arco/
   ```

4. Trigger manual library scan:
   ```bash
   curl -X POST "http://audiobookshelf.audiobookshelf.svc.cluster.local/api/libraries/<library_id>/scan" \
        -H "Authorization: Bearer <api_key>"
   ```

### Title mismatch (recording skipped)

Check logs for the expected vs actual title:
```bash
kubectl logs -n radigo job/<job-name> | grep -i title
```

If the program is on hiatus or replaced, this is expected behavior.

### Pod stuck in Pending

Usually PVC mounting issue:
```bash
kubectl describe pod -n radigo <pod-name>
```

Check if audiobookshelf pod is running on the same node (required for RWO PVC).

## Monitoring

### View recent recordings

```bash
# List jobs by completion time
kubectl get jobs -n radigo --sort-by=.status.completionTime

# Get last 5 job logs
for job in $(kubectl get jobs -n radigo -o jsonpath='{.items[-5:].metadata.name}'); do
  echo "=== $job ==="
  kubectl logs -n radigo job/$job --tail=20
done
```

### Check CronJob schedules

```bash
kubectl get cronjobs -n radigo -o wide
```
