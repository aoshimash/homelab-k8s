# Audiobookshelf Podcast Server

**Feature**: 005-audiobookshelf-podcast  
**Date**: 2026-01-03

## Overview

Audiobookshelf is a self-hosted podcast server deployed on the Kubernetes cluster. It provides web-based access to podcast libraries, RSS feed subscription, and automatic episode downloading. The service is exposed via Tailscale Ingress for secure tailnet-only access.

## Architecture

### Components

- **Namespace**: `audiobookshelf` - Isolates Audiobookshelf resources
- **Deployment**: Single replica running `ghcr.io/advplyr/audiobookshelf:latest`
- **Service**: ClusterIP service exposing port 80
- **Ingress**: Tailscale Ingress with hostname `audiobookshelf`
- **Storage**: Two PersistentVolumeClaims:
  - `audiobookshelf-config` (1Gi) - SQLite database and configuration
  - `audiobookshelf-podcasts` (50Gi) - Podcast audio files

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

Audiobookshelf is deployed via Flux CD from Git:

```bash
# Check Flux Kustomization status
flux get kustomizations apps

# Verify deployment
kubectl get all -n audiobookshelf
```

### Manual Verification

```bash
# Check pod status
kubectl get pods -n audiobookshelf

# Check PVCs
kubectl get pvc -n audiobookshelf

# Check Ingress
kubectl get ingress -n audiobookshelf
```

## Configuration

### Storage

- **Config volume** (`/config`): SQLite database, metadata cache, server configuration
- **Podcasts volume** (`/podcasts`): Podcast library directory

### Environment Variables

- `AUDIOBOOKSHELF_UID`: `99` (nobody user)
- `AUDIOBOOKSHELF_GID`: `99` (nobody group)

### Resource Limits

- **CPU**: Request 100m, Limit 1000m
- **Memory**: Request 256Mi, Limit 1Gi

### Health Checks

- **Liveness probe**: `/healthcheck` endpoint, 30s initial delay, 30s period
- **Readiness probe**: `/healthcheck` endpoint, 10s initial delay, 10s period

## Access

### Web Interface

Access from any device on your tailnet:

```
https://audiobookshelf.<your-tailnet>.ts.net
```

### First-Time Setup

1. **Create admin account** on first access
2. **Add podcast library**:
   - Settings → Libraries → Add Library
   - Library Type: Podcast
   - Folder: `/podcasts`

## Operations

### View Logs

```bash
kubectl logs -f -n audiobookshelf -l app=audiobookshelf
```

### Restart Pod

```bash
kubectl rollout restart deployment/audiobookshelf -n audiobookshelf
```

### Force Flux Reconciliation

```bash
flux reconcile kustomization apps --with-source
```

### Port Forward (Local Testing)

```bash
kubectl port-forward -n audiobookshelf svc/audiobookshelf 8080:80
# Access at http://localhost:8080
```

## Troubleshooting

### Pod Not Starting

```bash
# Check pod events
kubectl describe pod -n audiobookshelf -l app=audiobookshelf

# Check logs
kubectl logs -n audiobookshelf -l app=audiobookshelf
```

**Common issues**:
- Image pull failures → Check network connectivity
- Volume mount failures → Verify PVCs are bound
- Health check failures → Check application logs

### Storage Issues

```bash
# Check Longhorn volumes
kubectl get volumes.longhorn.io -n longhorn-system

# Check PVC events
kubectl describe pvc -n audiobookshelf audiobookshelf-podcasts
```

**Common issues**:
- PVC stuck in Pending → Check Longhorn storage availability
- Storage full → Increase PVC size or clean up old episodes

### Ingress Not Working

```bash
# Check ProxyGroup
kubectl get proxygroup -n tailscale ingress-proxies -o yaml

# Check Tailscale Operator logs
kubectl logs -n tailscale -l app.kubernetes.io/name=tailscale-operator
```

**Common issues**:
- Hostname not resolving → Check Tailscale admin console
- TLS not provisioning → Verify Tailscale Operator is running
- Access denied → Check Tailscale ACL policies

### Cannot Access from Tailnet

1. Verify device is on the same tailnet
2. Check Tailscale admin console for `audiobookshelf` device
3. Review ACL policies in Tailscale admin
4. Verify ProxyGroup is healthy

## Monitoring

### Grafana Alloy Integration

If Grafana Alloy is deployed:

- **Metrics**: Available in Grafana Cloud (filter by `namespace="audiobookshelf"`)
- **Logs**: Searchable in Loki (filter by `{namespace="audiobookshelf"}`)

### Health Check Endpoint

```bash
# From within cluster
kubectl exec -n audiobookshelf deploy/audiobookshelf -- wget -qO- localhost:80/healthcheck

# Via Tailscale
curl https://audiobookshelf.<tailnet>.ts.net/healthcheck
```

## Backup and Recovery

### Data Backup

Audiobookshelf data is stored in persistent volumes:

- **Config**: `/config` volume (1Gi) - Database and configuration
- **Podcasts**: `/podcasts` volume (50Gi) - Audio files

Backup strategies:

1. **Longhorn snapshots**: Use Longhorn UI to create volume snapshots
2. **PVC backup**: Export PVC data using `kubectl cp` or backup tools
3. **Application export**: Use Audiobookshelf's built-in export features

### Recovery

1. Restore from Longhorn snapshot
2. Or restore PVC from backup
3. Restart pod to pick up restored data

## Upgrades

### Container Image Updates

The deployment uses `latest` tag. To update:

1. Pull new image: `kubectl set image deployment/audiobookshelf audiobookshelf=ghcr.io/advplyr/audiobookshelf:latest -n audiobookshelf`
2. Or update deployment manifest and commit to Git (Flux will reconcile)

### Version Pinning

For production stability, consider pinning to a specific version:

```yaml
image: ghcr.io/advplyr/audiobookshelf:v2.7.0
```

## Security

- **Access**: Tailnet-only via Tailscale Ingress
- **TLS**: Automatically provisioned by Tailscale
- **User accounts**: Managed within Audiobookshelf application
- **Container**: Runs as non-root user (UID/GID 99)

## References

- [Audiobookshelf Documentation](https://www.audiobookshelf.org/)
- [GitHub Repository](https://github.com/advplyr/audiobookshelf)
- [Container Image](https://github.com/advplyr/audiobookshelf/pkgs/container/audiobookshelf)
