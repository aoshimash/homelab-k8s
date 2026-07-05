# Trilium Notes

## Overview

Trilium Notes (TriliumNext fork) is a self-hosted, hierarchical note-taking application deployed on the Kubernetes cluster. It runs as a sync server for Trilium's desktop and mobile clients, exposed for tailnet-only access via Tailscale Ingress.

## Architecture

### Components

- **Namespace**: `trilium` - Isolates Trilium resources
- **Deployment**: Single replica running `ghcr.io/triliumnext/trilium:v0.103.0`, `Recreate` update strategy (SQLite is single-writer, so two pods must never run against the same volume at once)
- **Service**: ClusterIP service exposing port 8080
- **Ingress**: Tailscale Ingress with hostname `trilium`
- **Storage**: `trilium-data` PersistentVolumeClaim (10Gi, Longhorn) holding the SQLite database and attachments; tagged with the `backup-daily` Longhorn recurring job since notes are non-recoverable data

### Access Flow

```
Tailnet Device → Tailscale ProxyGroup → Ingress → Service → Pod
```

## Deployment

### Prerequisites

- Flux CD running and reconciling
- Longhorn storage class available
- Tailscale Operator deployed
- ProxyGroup `ingress-proxies` exists

### GitOps Deployment

Trilium is deployed via Flux CD from Git — no manual steps are required after merge:

```bash
# Check Flux Kustomization status
flux get kustomizations apps

# Verify deployment
kubectl get pods,pvc,ingress -n trilium
```

## Configuration

### Container Image

- The published image is `ghcr.io/triliumnext/trilium:v0.103.0` (pinned).
- **Do not use `latest`**: upstream discourages it because an unplanned minor upgrade can break sync-protocol compatibility with already-paired desktop/mobile clients.
- Upstream documents `rootless` / `rootless-alpine` tags, but as of this writing they are not actually published (404 on both Docker Hub and ghcr.io). The standard image's entrypoint starts as root and drops privileges to `USER_UID`/`USER_GID` at startup — the same constraint as `paperless-ngx`, mitigated the same way (NOTE comment, `fsGroup`, `seccompProfile`).
- Image tags are **not** managed by Renovate (`k8s/apps/` is out of its scope today). Bump the tag manually in `k8s/apps/trilium/app/deployment.yaml` and commit.

### Storage

- **Data volume** (`/home/node/trilium-data`): SQLite database and file attachments. No external database is used.

### Environment Variables

- `TRILIUM_DATA_DIR`: `/home/node/trilium-data`
- `USER_UID` / `USER_GID`: `1000` — the UID/GID the entrypoint drops root privileges to
- `TZ`: `Asia/Tokyo` — ensures daily-note date boundaries roll over at JST midnight

### Resource Limits

- **CPU**: Request 100m, Limit 1000m
- **Memory**: Request 256Mi, Limit 1Gi

### Health Checks

- **Liveness probe**: `/api/health-check`, 30s initial delay, 30s period
- **Readiness probe**: `/api/health-check`, 10s initial delay, 10s period

`/api/health-check` returns `{"status":"ok"}` without authentication, so it's safe to use unauthenticated for probes.

## Access

### Web Interface / First-Time Setup

Access from any device on your tailnet:

```
https://trilium.<your-tailnet>.ts.net
```

No Secret or ConfigMap is provisioned — the initial password is set through the browser setup screen on first access.

### Desktop Client Sync

Trilium's desktop and mobile apps sync against a server instance. Point the desktop app's sync server URL at `https://trilium.<your-tailnet>.ts.net` — only devices on the tailnet can reach it.

## Operations

### View Logs

```bash
kubectl logs -f -n trilium -l app=trilium
```

### Restart Pod

```bash
kubectl rollout restart deployment/trilium -n trilium
```

### Force Flux Reconciliation

```bash
flux reconcile kustomization apps --with-source
```

### Image Updates

Updates are manual (same as `paperless-ngx`):

1. Check the [Trilium releases page](https://github.com/TriliumNext/Trilium/releases) for the latest tagged version.
2. Update the `image:` tag in `k8s/apps/trilium/app/deployment.yaml`.
3. Commit and push — Flux reconciles automatically. Never set the tag to `latest`.

## Troubleshooting

### Pod Not Starting

```bash
# Check pod events
kubectl describe pod -n trilium -l app=trilium

# Check logs
kubectl logs -n trilium -l app=trilium
```

**Common issues**:
- Image pull failures → check network connectivity / tag exists on ghcr.io
- Volume mount failures → verify the PVC is bound
- Health check failures → check application logs

### Storage Issues

```bash
# Check Longhorn volumes
kubectl get volumes.longhorn.io -n longhorn-system

# Check PVC events and backup annotation
kubectl get pvc trilium-data -n trilium -o jsonpath='{.metadata.annotations}'
```

### Ingress Not Working

```bash
# Check ProxyGroup
kubectl get proxygroup -n tailscale ingress-proxies -o yaml

# Check Tailscale Operator logs
kubectl logs -n tailscale -l app.kubernetes.io/name=tailscale-operator
```

## Verification

```bash
# Flux Kustomization is Ready
flux get kustomizations apps

# Deployment/Service/Ingress/PVC exist with no manual steps
kubectl get pods,pvc,ingress -n trilium

# PVC carries the backup-daily annotation
kubectl get pvc trilium-data -n trilium -o jsonpath='{.metadata.annotations}'

# Health check responds (from within cluster)
kubectl exec -n trilium deploy/trilium -- wget -qO- localhost:8080/api/health-check

# Reachable via Tailscale and shows the initial setup screen
curl https://trilium.<tailnet>.ts.net/api/health-check
```

## Backup and Recovery

Trilium data lives entirely in the `trilium-data` PVC (10Gi):

- Tagged with the `backup-daily` Longhorn recurring job for automatic daily snapshots.
- To restore, recover the volume from a Longhorn snapshot and restart the pod to pick up the restored data.

## Security

- **Access**: Tailnet-only via Tailscale Ingress
- **TLS**: Automatically provisioned by Tailscale
- **Authentication**: Built-in password auth, set via the browser on first access
- **Container**: Entrypoint starts as root and drops to UID/GID 1000 (rootless image tags are not yet published upstream)

## Out of Scope

- Migrating to a rootless image (revisit once upstream actually publishes `rootless` tags)
- Bringing `k8s/apps/` image tags under Renovate management
- Importing existing note data (fresh setup assumed)

## References

- [Trilium Notes (TriliumNext) GitHub Repository](https://github.com/TriliumNext/Trilium)
- [Docker Installation Docs](https://docs.triliumnotes.org/user-guide/setup/server/installation/docker)
- [Container Image](https://github.com/triliumnext/trilium/pkgs/container/trilium)
