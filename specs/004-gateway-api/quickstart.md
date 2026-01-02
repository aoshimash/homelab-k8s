# Quickstart: Gateway API CRD Installation

**Feature**: 004-gateway-api
**Date**: 2026-01-03

## Prerequisites

- Kubernetes cluster running (Talos Linux)
- Flux CD installed and operational
- kubectl configured with cluster access
- Git repository cloned locally

## Step 1: Create Gateway API Directory

```bash
mkdir -p k8s/infrastructure/gateway-api
```

## Step 2: Create GitRepository Resource

```bash
cat > k8s/infrastructure/gateway-api/gitrepository.yaml << 'EOF'
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
EOF
```

## Step 3: Create Flux Kustomization for CRDs

```bash
cat > k8s/infrastructure/gateway-api/kustomization-crds.yaml << 'EOF'
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: gateway-api-crds
  namespace: flux-system
spec:
  interval: 10m0s
  path: ./config/crd/standard
  prune: false
  sourceRef:
    kind: GitRepository
    name: gateway-api
    namespace: flux-system
EOF
```

## Step 4: Create Local Kustomization

```bash
cat > k8s/infrastructure/gateway-api/kustomization.yaml << 'EOF'
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - gitrepository.yaml
  - kustomization-crds.yaml
EOF
```

## Step 5: Update Infrastructure Kustomization

Edit `k8s/infrastructure/kustomization.yaml` to add gateway-api as the first resource:

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - gateway-api/    # Add this line first
  - cilium/
  - longhorn/
  - tailscale/
```

## Step 6: Commit and Push Changes

```bash
git add k8s/infrastructure/gateway-api/
git add k8s/infrastructure/kustomization.yaml
git commit -m "feat: add Gateway API CRD installation via Flux"
git push
```

## Step 7: Verify Installation

Wait for Flux to reconcile (or trigger manually):

```bash
# Trigger reconciliation (optional)
flux reconcile source git flux-system
flux reconcile kustomization infrastructure

# Check GitRepository status
kubectl get gitrepository gateway-api -n flux-system

# Check Kustomization status
kubectl get kustomization gateway-api-crds -n flux-system

# Verify CRDs are installed
kubectl get crd | grep gateway.networking.k8s.io
```

Expected CRDs:
- `gatewayclasses.gateway.networking.k8s.io`
- `gateways.gateway.networking.k8s.io`
- `grpcroutes.gateway.networking.k8s.io`
- `httproutes.gateway.networking.k8s.io`
- `referencegrants.gateway.networking.k8s.io`

## Step 8: Verify Tailscale Gateway

After CRDs are installed, verify the existing Tailscale Gateway becomes operational:

```bash
# Check Gateway status
kubectl get gateway tailscale-gateway -n tailscale

# Check HTTPRoute status
kubectl get httproute longhorn-ui -n longhorn-system

# Verify Gateway is programmed
kubectl get gateway tailscale-gateway -n tailscale -o jsonpath='{.status.conditions[?(@.type=="Programmed")].status}'
```

## Step 9: Test Longhorn UI Access

From a device connected to your Tailnet:

1. Open browser
2. Navigate to `https://longhorn.<your-tailnet>.ts.net`
3. Verify Longhorn UI dashboard loads

## Troubleshooting

### CRDs not appearing

```bash
# Check GitRepository status
kubectl describe gitrepository gateway-api -n flux-system

# Check Kustomization events
kubectl describe kustomization gateway-api-crds -n flux-system
```

### Gateway not becoming ready

```bash
# Check Tailscale Operator logs
kubectl logs -n tailscale -l app.kubernetes.io/name=tailscale-operator

# Check Gateway events
kubectl describe gateway tailscale-gateway -n tailscale
```

### HTTPRoute not accepted

```bash
# Check HTTPRoute status
kubectl describe httproute longhorn-ui -n longhorn-system

# Verify parent Gateway reference
kubectl get httproute longhorn-ui -n longhorn-system -o yaml
```

## Cleanup (if needed)

To remove Gateway API CRDs (WARNING: this will delete all Gateway/HTTPRoute resources):

```bash
# Remove from infrastructure
# Edit k8s/infrastructure/kustomization.yaml to remove gateway-api/

# Delete the directory
rm -rf k8s/infrastructure/gateway-api/

# Commit and push
git add -A
git commit -m "chore: remove Gateway API CRDs"
git push

# Manually delete CRDs if needed (Flux won't prune them)
kubectl delete crd gatewayclasses.gateway.networking.k8s.io
kubectl delete crd gateways.gateway.networking.k8s.io
kubectl delete crd httproutes.gateway.networking.k8s.io
kubectl delete crd grpcroutes.gateway.networking.k8s.io
kubectl delete crd referencegrants.gateway.networking.k8s.io
```
