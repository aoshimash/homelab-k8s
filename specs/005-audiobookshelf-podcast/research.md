# Research: Audiobookshelf Podcast Server

**Feature**: 005-audiobookshelf-podcast
**Date**: 2026-01-03

## Research Topics

### 1. Audiobookshelf Container Image

**Decision**: Use `ghcr.io/advplyr/audiobookshelf:latest`

**Rationale**:
- Official image maintained by the Audiobookshelf developer
- GitHub Container Registry is reliable and fast
- The `latest` tag is acceptable for homelab use; consider pinning to a specific version for production stability

**Alternatives Considered**:
- `advplyr/audiobookshelf` (Docker Hub) - Valid alternative, same image
- Self-built image - Unnecessary complexity for a well-maintained project

### 2. Audiobookshelf Required Volumes

**Decision**: Two PersistentVolumeClaims required

| Mount Path | Purpose | Size | Access Mode |
|------------|---------|------|-------------|
| `/config` | SQLite database, metadata cache, server configuration | 1Gi | ReadWriteOnce |
| `/podcasts` | Podcast audio files downloaded from RSS feeds | 50Gi | ReadWriteOnce |

**Rationale**:
- Audiobookshelf uses SQLite for its database, stored in `/config`
- Podcast episodes are stored in library folders; `/podcasts` is the designated podcast library
- Separating config from media allows independent backup strategies
- 50Gi for podcasts based on clarification session

**Alternatives Considered**:
- Single volume for everything - Less flexible for backup/restore
- `/audiobooks` volume - Not needed for podcast-only use case

### 3. Audiobookshelf Container Configuration

**Decision**: Use environment variables and volume mounts

| Environment Variable | Value | Purpose |
|---------------------|-------|---------|
| `AUDIOBOOKSHELF_UID` | `99` | Run as nobody user |
| `AUDIOBOOKSHELF_GID` | `99` | Run as nobody group |

**Rationale**:
- UID/GID 99 (nobody) is a common pattern for non-root containers
- Audiobookshelf automatically creates the podcast library on first run
- Initial admin user is created through the web UI on first access

**Alternatives Considered**:
- Root user (UID 0) - Security concern, unnecessary
- Custom UID - No benefit over standard nobody

### 4. Tailscale Ingress Configuration

**Decision**: Use existing ProxyGroup `ingress-proxies` with Tailscale Ingress

**Configuration**:
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: audiobookshelf
  namespace: audiobookshelf
  annotations:
    tailscale.com/proxy-group: ingress-proxies
spec:
  ingressClassName: tailscale
  defaultBackend:
    service:
      name: audiobookshelf
      port:
        number: 80
  tls:
    - hosts:
        - audiobookshelf
```

**Rationale**:
- Follows the same pattern as `longhorn-ui` Ingress in `k8s/configs/tailscale/`
- ProxyGroup `ingress-proxies` already exists and configured with 1 replica
- TLS is automatically provisioned by Tailscale
- Hostname `audiobookshelf` results in `audiobookshelf.<tailnet>.ts.net`

**Alternatives Considered**:
- Individual proxy per service - Wastes resources in single-node cluster
- ClusterIP only - Requires VPN or port-forward for access

### 5. Service Port Configuration

**Decision**: Service exposes port 80, container listens on 80

| Port | Type | Purpose |
|------|------|---------|
| 80 | HTTP | Web UI and API |

**Rationale**:
- Audiobookshelf listens on port 80 by default
- TLS termination happens at Tailscale Ingress level
- No need for HTTPS at the service level within the cluster

### 6. Flux Kustomization for Apps

**Decision**: Create new `apps-kustomization.yaml` for user applications

**Rationale**:
- Constitution defines `k8s/apps/` for user applications
- Separate Kustomization allows independent reconciliation
- Dependencies can be set to ensure infrastructure is ready first

**Configuration**:
```yaml
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: apps
  namespace: flux-system
spec:
  interval: 10m0s
  path: ./k8s/apps
  prune: true
  sourceRef:
    kind: GitRepository
    name: flux-system
    namespace: flux-system
  dependsOn:
    - name: configs
```

### 7. Health Check Configuration

**Decision**: Use HTTP liveness and readiness probes

**Configuration**:
```yaml
livenessProbe:
  httpGet:
    path: /healthcheck
    port: 80
  initialDelaySeconds: 30
  periodSeconds: 30
readinessProbe:
  httpGet:
    path: /healthcheck
    port: 80
  initialDelaySeconds: 10
  periodSeconds: 10
```

**Rationale**:
- Audiobookshelf provides a `/healthcheck` endpoint
- Readiness probe ensures traffic is only sent when the app is ready
- Liveness probe restarts the pod if the app becomes unresponsive

### 8. Resource Requirements

**Decision**: Set reasonable resource requests and limits

| Resource | Request | Limit |
|----------|---------|-------|
| CPU | 100m | 1000m |
| Memory | 256Mi | 1Gi |

**Rationale**:
- Audiobookshelf is a Node.js application with moderate resource requirements
- Transcoding (if enabled) may require more CPU; limits allow bursting
- Memory limit prevents runaway processes from affecting cluster stability

## Summary

All technical decisions have been made based on Audiobookshelf documentation, existing cluster patterns, and homelab best practices. No NEEDS CLARIFICATION items remain.
