# Quickstart: Migrate radigo to yt-dlp-rajiko

**Feature**: 015-migrate-radigo-yt-dlp  
**Date**: 2026-01-31

## Prerequisites

- Access to homelab Kubernetes cluster
- `kubectl` configured for cluster access
- Docker for building images
- GitHub Container Registry access (ghcr.io)

## Quick Test (Local)

Test yt-dlp-rajiko locally before deploying:

```bash
# Install yt-dlp with rajiko plugin
pip install yt-dlp yt-dlp-rajiko

# Test timefree download (replace with recent broadcast)
yt-dlp -v --embed-metadata \
  "https://radiko.jp/#!/ts/TBS/20260128010000"
```

## Build and Push Image

```bash
# Build the new Docker image
cd images/yt-dlp-rajiko
docker build -t ghcr.io/aoshimash/homelab-k8s/yt-dlp-rajiko:v1.0.0 .

# Push to registry
docker push ghcr.io/aoshimash/homelab-k8s/yt-dlp-rajiko:v1.0.0
```

## Deploy via GitOps

All changes are applied via Git commit - Flux handles reconciliation:

```bash
# Commit changes
git add k8s/apps/audiobookshelf/radigo-recorder/
git add images/yt-dlp-rajiko/
git rm -r images/radigo/
git commit -m "feat: migrate radigo to yt-dlp-rajiko"

# Push to trigger Flux reconciliation
git push origin 015-migrate-radigo-yt-dlp
```

## Manual Testing

### Test Recording Job

```bash
# Create a one-time test job from the CronJob
kubectl create job --from=cronjob/radigo-ijuin test-ijuin -n audiobookshelf

# Watch job progress
kubectl logs -f job/test-ijuin -n audiobookshelf

# Check output
kubectl exec -it deployment/audiobookshelf -n audiobookshelf -- \
  ls -la /podcasts/ijuin/
```

### Verify Recording

```bash
# Check file exists and has content
kubectl exec -it deployment/audiobookshelf -n audiobookshelf -- \
  ls -lh /podcasts/ijuin/*.m4a

# Verify metadata (if ffprobe available)
kubectl exec -it deployment/audiobookshelf -n audiobookshelf -- \
  ffprobe /podcasts/ijuin/*.m4a
```

## Rollback

If issues occur, revert the Git commit:

```bash
git revert HEAD
git push origin 015-migrate-radigo-yt-dlp
```

Flux will reconcile back to previous state.

## Key Files Changed

| File | Change |
|------|--------|
| `images/yt-dlp-rajiko/Dockerfile` | NEW |
| `images/radigo/Dockerfile` | DELETED |
| `k8s/apps/audiobookshelf/radigo-recorder/configmap-record-script.yaml` | UPDATED |
| `k8s/apps/audiobookshelf/radigo-recorder/base/cronjob.yaml` | UPDATED |

## Troubleshooting

### Job Fails Immediately
```bash
kubectl describe job/test-ijuin -n audiobookshelf
kubectl logs job/test-ijuin -n audiobookshelf
```

### yt-dlp Plugin Not Found
Verify image contains plugin:
```bash
kubectl run test-ytdlp --rm -it --image=ghcr.io/aoshimash/homelab-k8s/yt-dlp-rajiko:v1.0.0 -- yt-dlp -v 2>&1 | grep -i rajiko
```

### Network/Radiko API Errors
- Check if radiko.jp is accessible from cluster
- Verify timefree URL format is correct
- Check job logs for specific error messages
