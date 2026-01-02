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
Configure Cilium to bypass socket load balancing for the `tailscale` namespace.

### Rationale
Per [Tailscale documentation](https://tailscale.com/kb/1236/kubernetes-operator):
> You must enable bypassing socket load balancer in Pods' namespaces if you run Cilium in kube-proxy replacement mode

This is required for:
- Tailscale LoadBalancer Services
- Services exposed via `tailscale.com/expose` annotation
- Service CIDR ranges via Connector

### Implementation
Add annotation to the tailscale namespace:
```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: tailscale
  labels:
    pod-security.kubernetes.io/enforce: privileged
  annotations:
    # Bypass Cilium socket LB for Tailscale proxy compatibility
    io.cilium.proxy-visibility: "<Egress/53/UDP/DNS>"
```

Or configure via Cilium CiliumNetworkPolicy or Helm values.

**Note**: The exact Cilium configuration may need to be determined based on Cilium version. Common approaches:
1. Namespace annotation: `cilium.io/global-service: "false"`
2. CiliumClusterwideNetworkPolicy for socket LB bypass
3. Cilium Helm values: `socketLB.hostNamespaceOnly: true`

### Alternatives Considered
- **Disable kube-proxy replacement entirely**: Rejected as it would require cluster reconfiguration
- **Use Tailscale without Ingress/LoadBalancer**: Rejected as Ingress is a primary use case

## 4. Tailscale Ingress Configuration

### Decision
Use Kubernetes Ingress resource with `ingressClassName: tailscale` to expose Longhorn UI.

### Rationale
- Native Kubernetes Ingress API
- Tailscale Operator watches for Ingress resources with `ingressClassName: tailscale`
- Automatic TLS certificate provisioning
- Device appears in Tailscale admin console

### Implementation
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: longhorn-ui
  namespace: longhorn-system
spec:
  ingressClassName: tailscale
  rules:
    - host: longhorn  # becomes longhorn.<tailnet>.ts.net
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: longhorn-frontend
                port:
                  number: 80
```

### Alternatives Considered
- **Tailscale Service annotation**: `tailscale.com/expose: "true"` - Simpler but less control
- **API Server Proxy**: Not applicable for web UI access

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
└── ingress-longhorn.yaml
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
| Cilium Compatibility | Configure socket LB bypass for tailscale namespace |
| Ingress Method | Kubernetes Ingress with ingressClassName: tailscale |
| Namespace | Dedicated `tailscale` namespace with privileged PSS |
| GitOps Pattern | Follow existing cilium/longhorn directory structure |
