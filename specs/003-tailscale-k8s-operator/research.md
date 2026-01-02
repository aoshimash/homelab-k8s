# Research: Tailscale Kubernetes Operator Installation

**Feature**: 003-tailscale-k8s-operator
**Date**: 2026-01-02

## 1. Tailscale Operator Helm Chart

### Decision
Use the official Tailscale Helm chart from `https://pkgs.tailscale.com/helmcharts` with a pinned version.

### Rationale
- Official chart maintained by Tailscale
- Supports all operator features (Ingress, Egress, API Server Proxy, etc.)
- Helm values provide configuration flexibility
- Version pinning ensures reproducibility per spec requirement FR-009

### Alternatives Considered
- **Static manifests from GitHub**: Rejected because Helm provides better configuration management and Flux HelmRelease integration
- **Manual deployment**: Rejected as it violates GitOps principles

### Chart Version Strategy
- Pin to a specific version (e.g., `1.76.1` or latest stable at deployment time)
- Version can be checked via `helm search repo tailscale/tailscale-operator --versions`
- Update by modifying HelmRelease spec.chart.spec.version

## 2. OAuth Credentials Configuration

### Decision
Store OAuth client ID and secret in a SOPS-encrypted Kubernetes Secret, referenced by the HelmRelease.

### Rationale
- Follows existing pattern from `longhorn/secret-r2-credentials.sops.yaml`
- Flux Kustomization already configured for SOPS decryption
- Keeps credentials encrypted at rest in Git repository

### Configuration Structure
```yaml
# secret-oauth.sops.yaml
apiVersion: v1
kind: Secret
metadata:
  name: tailscale-operator-oauth
  namespace: tailscale
type: Opaque
stringData:
  client_id: <encrypted>
  client_secret: <encrypted>
```

### HelmRelease Reference
```yaml
values:
  oauth:
    clientId:
      valueFrom:
        secretKeyRef:
          name: tailscale-operator-oauth
          key: client_id
    clientSecret:
      valueFrom:
        secretKeyRef:
          name: tailscale-operator-oauth
          key: client_secret
```

**Note**: The Tailscale Helm chart expects `oauth.clientId` and `oauth.clientSecret` as direct values. For secret reference, use `existingSecret` pattern if supported, or create the secret with the expected name `operator-oauth` and let the chart reference it automatically.

### Alternatives Considered
- **Direct values in HelmRelease**: Rejected as it would expose credentials in Git
- **External secrets operator**: Rejected as overkill for homelab; SOPS is already in use

## 3. Cilium Compatibility (kube-proxy replacement mode)

### Decision
**No special configuration required** for Gateway API / HTTPRoute usage.

### Rationale
Per [Tailscale documentation](https://tailscale.com/kb/1236/kubernetes-operator):
> You must enable bypassing socket load balancer in Pods' namespaces if you run Cilium in kube-proxy replacement mode

This is **only required for**:
- Tailscale LoadBalancer Services
- Services exposed via `tailscale.com/expose` annotation
- Service CIDR ranges via Connector

**Gateway API / HTTPRoute does NOT require this configuration** because:
- The Tailscale proxy runs inside the cluster and connects to ClusterIP services
- No external LoadBalancer IP allocation is needed
- Traffic flows: Tailnet → Tailscale Proxy Pod → ClusterIP Service

### Implementation
No additional Cilium configuration needed. Standard namespace with privileged PSS is sufficient.

### Alternatives Considered
- **Configure socket LB bypass anyway**: Rejected as unnecessary complexity for Gateway API use case
- **Use LoadBalancer Service**: Rejected - Gateway API is preferred for HTTP services

## 4. Tailscale Gateway API Configuration

### Decision
Use Kubernetes Gateway API (Gateway + HTTPRoute) instead of legacy Ingress API to expose Longhorn UI.

### Rationale
- **Future-proof**: Gateway API is the successor to Ingress API with active development
- **Ingress API status**: Stable (GA) but no new features; new functionality goes to Gateway API
- **Tailscale support**: Operator supports both `ingressClassName: tailscale` and `gatewayClassName: tailscale`
- **Better separation**: Gateway (infra) and HTTPRoute (app) are separate concerns
- **Richer features**: More expressive routing, traffic management capabilities

### Implementation

**Gateway (created once, in tailscale namespace)**:
```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: tailscale-gateway
  namespace: tailscale
spec:
  gatewayClassName: tailscale
  listeners:
    - name: https
      protocol: HTTPS
      port: 443
```

**HTTPRoute (per service, in service's namespace)**:
```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: longhorn-ui
  namespace: longhorn-system
spec:
  parentRefs:
    - name: tailscale-gateway
      namespace: tailscale
  hostnames:
    - "longhorn"  # becomes longhorn.<tailnet>.ts.net
  rules:
    - matches:
        - path:
            type: PathPrefix
            value: /
      backendRefs:
        - name: longhorn-frontend
          port: 80
```

### Comparison: Ingress vs Gateway API

| Aspect | Ingress | Gateway API |
|--------|---------|-------------|
| Resources | 1 (Ingress) | 2 (Gateway + HTTPRoute) |
| Future | Maintenance mode | Active development |
| Complexity | Simple | Slightly more complex |
| Tailscale support | ✅ | ✅ |
| Recommended | For simple cases | For new deployments |

### Alternatives Considered
- **Kubernetes Ingress API**: Rejected - Gateway API is more future-proof
- **Tailscale Service annotation**: `tailscale.com/expose: "true"` - Rejected, less control and L4 only

## 5. Namespace Configuration

### Decision
Create dedicated `tailscale` namespace with appropriate Pod Security Standards.

### Rationale
- Operator requires privileged containers for network configuration
- Separation from other infrastructure components
- Follows Tailscale documentation recommendations

### Implementation
```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: tailscale
  labels:
    pod-security.kubernetes.io/enforce: privileged
    pod-security.kubernetes.io/audit: privileged
    pod-security.kubernetes.io/warn: privileged
```

## 6. Flux Integration Pattern

### Decision
Follow existing pattern from `cilium/` and `longhorn/` directories.

### Rationale
- Consistency with existing infrastructure
- Proven pattern in this repository
- SOPS decryption already configured in infrastructure Kustomization

### File Structure
```
k8s/infrastructure/tailscale/
├── kustomization.yaml
├── namespace.yaml
├── helmrepository.yaml
├── helmrelease.yaml
├── secret-oauth.sops.yaml
├── gateway.yaml              # Gateway API - Tailscale gateway
└── httproute-longhorn.yaml   # Gateway API - HTTPRoute for Longhorn UI
```

### Integration Points
1. Add `tailscale/` to `k8s/infrastructure/kustomization.yaml`
2. Flux infrastructure Kustomization already handles SOPS decryption
3. HelmRelease references HelmRepository in flux-system namespace

## Summary of Decisions

| Topic | Decision |
|-------|----------|
| Helm Chart Source | Official Tailscale chart from pkgs.tailscale.com |
| Version Strategy | Pin specific version for reproducibility |
| OAuth Credentials | SOPS-encrypted Secret |
| Cilium Compatibility | No special config needed for Gateway API |
| Routing Method | Kubernetes Gateway API (Gateway + HTTPRoute) |
| Namespace | Dedicated `tailscale` namespace with privileged PSS |
| GitOps Pattern | Follow existing cilium/longhorn directory structure |
