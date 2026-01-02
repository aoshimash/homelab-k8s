# Research: Gateway API CRD Installation

**Feature**: 004-gateway-api
**Date**: 2026-01-03

## 1. Gateway API CRD Installation Method

### Decision
Use Flux GitRepository + Kustomization to install Gateway API CRDs from the official kubernetes-sigs/gateway-api repository.

### Rationale
- GitOps-compliant approach using Flux
- Version pinning via Git tag reference
- Automatic reconciliation and updates
- No need to commit large CRD files to the repository

### Implementation

**GitRepository (references upstream repo)**:
```yaml
apiVersion: source.toolkit.fluxcd.io/v1
kind: GitRepository
metadata:
  name: gateway-api
  namespace: flux-system
spec:
  interval: 1h
  url: https://github.com/kubernetes-sigs/gateway-api
  ref:
    tag: v1.4.1
```

**Kustomization (installs CRDs from the repo)**:
```yaml
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: gateway-api-crds
  namespace: flux-system
spec:
  interval: 10m0s
  path: ./config/crd/standard
  prune: false  # CRDs should not be pruned
  sourceRef:
    kind: GitRepository
    name: gateway-api
    namespace: flux-system
```

### Alternatives Considered
- **kubectl apply with remote URL**: Rejected - violates GitOps principles
- **Download and commit CRD YAML**: Rejected - large files (~700KB), harder to update
- **OCI artifact**: Not officially provided by kubernetes-sigs/gateway-api

## 2. Standard vs Experimental Channel

### Decision
Use the **standard channel** CRDs (`config/crd/standard`).

### Rationale
- Standard channel contains GA (v1) resources only
- Experimental channel CRDs are too large (>1MB) and cause `kubectl apply` issues
- Tailscale Operator only requires GA resources (Gateway, HTTPRoute, GatewayClass)
- Production stability is preferred over experimental features

### Standard Channel Contents (v1.4.1)
- GatewayClass
- Gateway
- HTTPRoute
- ReferenceGrant
- GRPCRoute (GA in v1.2.0+)

### Experimental Channel Additional Resources
- TCPRoute
- TLSRoute
- UDPRoute
- BackendTLSPolicy
- BackendLBPolicy

## 3. CRD Pruning Strategy

### Decision
Set `prune: false` for the Gateway API CRD Kustomization.

### Rationale
- CRDs are cluster-scoped resources that should persist
- Pruning CRDs would delete all instances of those resources
- Standard Flux behavior for CRD management
- Matches existing patterns (Cilium CRDs are not managed by Flux either)

### Implementation Note
The `prune: false` setting ensures that if the Kustomization is deleted or the source changes, the CRDs remain in the cluster. This is the recommended approach for CRD management.

## 4. Resource Ordering in Infrastructure

### Decision
Add `gateway-api/` as the first entry in `k8s/infrastructure/kustomization.yaml`.

### Rationale
- CRDs must be available before resources that use them
- Kustomize applies resources in order listed
- No explicit `dependsOn` needed for cluster-level CRDs
- Simpler configuration, matches existing patterns

### Implementation
```yaml
# k8s/infrastructure/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - gateway-api/    # CRDs first
  - cilium/
  - longhorn/
  - tailscale/
```

## 5. Cross-Namespace Reference for HTTPRoute

### Decision
Use Gateway's `allowedRoutes` configuration instead of ReferenceGrant.

### Rationale
- HTTPRoute in `longhorn-system` namespace needs to reference Gateway in `tailscale` namespace
- Gateway API provides two mechanisms:
  1. Gateway `allowedRoutes.namespaces.from: All` - simpler
  2. ReferenceGrant in target namespace - more granular control
- For homelab use case, allowing all namespaces is acceptable
- Tailscale Operator may handle this automatically

### Implementation
The existing Gateway configuration already allows cross-namespace routes:
```yaml
spec:
  gatewayClassName: tailscale
  listeners:
    - name: https
      protocol: HTTPS
      port: 443
      # allowedRoutes defaults to Same namespace, but Tailscale operator
      # may override this behavior
```

### Note
If cross-namespace routing doesn't work automatically, add explicit configuration:
```yaml
listeners:
  - name: https
    allowedRoutes:
      namespaces:
        from: All
```

## 6. Directory Structure

### Decision
Create `k8s/infrastructure/gateway-api/` directory with Flux resources.

### File Structure
```
k8s/infrastructure/gateway-api/
├── kustomization.yaml      # Local Kustomize config
├── gitrepository.yaml      # Flux GitRepository for upstream
└── kustomization-crds.yaml # Flux Kustomization for CRD installation
```

### Rationale
- Follows existing pattern from cilium/, longhorn/, tailscale/
- Separates CRD management from application resources
- Clear organization for GitOps workflow

## Summary of Decisions

| Topic | Decision |
|-------|----------|
| Installation Method | Flux GitRepository + Kustomization |
| CRD Channel | Standard (not experimental) |
| Version | v1.4.1 (pinned via Git tag) |
| Pruning | Disabled (`prune: false`) |
| Resource Order | gateway-api/ first in infrastructure |
| Cross-namespace | Gateway allowedRoutes or Tailscale default |
| Directory | k8s/infrastructure/gateway-api/ |
