# Research: Tailscale Kubernetes Operator

**Feature**: 003-tailscale-k8s-operator
**Date**: 2026-01-03

## Research Topics

### 1. Tailscale Kubernetes Operator Installation

**Decision**: Use Helm chart from official Tailscale repository via Flux HelmRelease

**Rationale**:
- Official Helm chart is maintained by Tailscale at `https://pkgs.tailscale.com/helmcharts`
- GitOps-compatible through Flux HelmRelease resource
- Follows existing pattern used for Cilium and Longhorn in this cluster

**Alternatives Considered**:
- Static manifests: Rejected due to less flexibility and harder version management
- Manual kubectl apply: Rejected per GitOps-First principle in constitution

**Key Configuration**:
- Helm repository: `https://pkgs.tailscale.com/helmcharts`
- Chart name: `tailscale-operator`
- Namespace: `tailscale` (official recommendation)
- OAuth credentials passed via `oauth.clientId` and `oauth.clientSecret` values

### 2. Cilium kube-proxy Replacement Compatibility

**Decision**: Configure Cilium to bypass socket load balancing for the `tailscale` namespace

**Rationale**:
- Per [Tailscale KB](https://tailscale.com/kb/1236/kubernetes-operator#cilium-in-kube-proxy-replacement-mode), when Cilium runs in kube-proxy replacement mode with socket LB enabled, connections from Pods to ClusterIPs bypass the veth device and thus bypass Tailscale firewall rules
- The solution is to configure Cilium's `socketLB.hostNamespaceOnly: true` or use namespace-based bypass

**Implementation**:
- Add `socketLB.hostNamespaceOnly: true` to Cilium HelmRelease values
- This ensures socket LB only applies in host namespace, not in Pod namespaces
- Alternative: Use CiliumNetworkPolicy to bypass specific namespaces

**Reference**: https://tailscale.com/kb/1236/kubernetes-operator#cilium-in-kube-proxy-replacement-mode

### 3. ProxyGroup for Ingress

**Decision**: Use ProxyGroup with 1 replica for resource efficiency on single-node cluster

**Rationale**:
- ProxyGroup (available in Tailscale Operator 1.76+) allows multiple Ingress resources to share proxy infrastructure
- Reduces resource consumption compared to per-service proxy pods
- Single replica is appropriate for single-node homelab cluster

**Implementation**:
- Create ProxyGroup resource with `type: ingress` and `replicas: 1`
- Ingress resources reference the ProxyGroup via `proxyGroup` field
- ProxyGroup creates a StatefulSet with the specified replica count

**Key Points**:
- ProxyGroup is a Tailscale CRD (`tailscale.com/v1alpha1`)
- Each replica becomes a separate tailnet device
- Ingress resources can optionally reference a ProxyGroup

### 4. Tailscale Ingress Configuration

**Decision**: Use standard Kubernetes Ingress with `ingressClassName: tailscale`

**Rationale**:
- Tailscale Operator watches for Ingress resources with `ingressClassName: tailscale`
- Automatic TLS certificate provisioning via Tailscale's HTTPS feature
- Hostname is derived from Ingress name or can be specified

**Implementation**:
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: longhorn-ui
  namespace: longhorn-system
spec:
  ingressClassName: tailscale
  rules:
    - host: longhorn  # becomes longhorn.<tailnet-name>.ts.net
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

### 5. OAuth Credential Management

**Decision**: Store OAuth credentials as SOPS-encrypted Kubernetes Secret

**Rationale**:
- Follows existing pattern for Longhorn R2 credentials
- Compliant with Secrets Management principle in constitution
- Flux decrypts secrets during reconciliation

**Implementation**:
- Create `secret-oauth-credentials.sops.yaml` with `oauth.clientId` and `oauth.clientSecret`
- Reference secret in HelmRelease via `existingSecret` or environment variable injection
- Alternative: Use `oauth.clientId` and `oauth.clientSecret` values with SOPS-encrypted values file

### 6. ACL Tag Strategy

**Decision**: Use `tag:k8s-operator` for operator, `tag:k8s` for proxy devices

**Rationale**:
- Simplified ACL management with generic tag
- Operator needs separate tag for management operations
- Proxy devices share common tag for uniform access control

**Implementation**:
- Configure OAuth client with tags: `tag:k8s-operator`, `tag:k8s`
- Set `operatorConfig.hostname` and tags in Helm values
- Proxy devices automatically tagged based on OAuth client configuration

### 7. Longhorn UI Service Discovery

**Decision**: Expose Longhorn UI service via Tailscale Ingress as smoke test

**Rationale**:
- Validates end-to-end Tailscale Ingress functionality
- Uses existing cluster service (no new deployments needed)
- Provides practical utility for cluster management

**Service Details**:
- Service name: `longhorn-frontend` in `longhorn-system` namespace
- Port: 80 (HTTP)
- Tailscale hostname: `longhorn.<tailnet>.ts.net`

## Unresolved Items

None - all technical decisions have been made.

## References

- [Tailscale Kubernetes Operator Documentation](https://tailscale.com/kb/1236/kubernetes-operator)
- [Tailscale Helm Charts](https://pkgs.tailscale.com/helmcharts)
- [Cilium kube-proxy Replacement Compatibility](https://tailscale.com/kb/1236/kubernetes-operator#cilium-in-kube-proxy-replacement-mode)
- [ProxyGroup Configuration](https://tailscale.com/kb/1236/kubernetes-operator#optional-pre-creating-a-proxygroup)
