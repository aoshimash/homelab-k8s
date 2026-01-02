# Quickstart: Tailscale Kubernetes Operator Installation

**Feature**: 003-tailscale-k8s-operator
**Date**: 2026-01-02

## Prerequisites

Before starting, ensure you have:

1. **Tailscale Account** with admin access
2. **OAuth Client** created in Tailscale admin console:
   - Navigate to: Settings → OAuth clients → Generate OAuth client
   - Scopes: `Devices Core`, `Auth Keys`, `Services` (all write)
   - Tags: `tag:k8s-operator`
   - Save the `client_id` and `client_secret`

3. **Local Tools**:
   - `kubectl` configured for your cluster
   - `sops` installed with age key configured
   - `flux` CLI (optional, for manual reconciliation)

4. **Cluster Requirements**:
   - Flux CD running and healthy
   - SOPS age key deployed as `sops-age` Secret in `flux-system`
   - Longhorn installed (for Ingress validation)

## Quick Deploy Steps

### Step 1: Create the Tailscale Directory

```bash
mkdir -p k8s/infrastructure/tailscale
```

### Step 2: Create Namespace

```bash
cat > k8s/infrastructure/tailscale/namespace.yaml << 'EOF'
apiVersion: v1
kind: Namespace
metadata:
  name: tailscale
  labels:
    pod-security.kubernetes.io/enforce: privileged
    pod-security.kubernetes.io/audit: privileged
    pod-security.kubernetes.io/warn: privileged
EOF
```

### Step 3: Create HelmRepository

```bash
cat > k8s/infrastructure/tailscale/helmrepository.yaml << 'EOF'
apiVersion: source.toolkit.fluxcd.io/v1
kind: HelmRepository
metadata:
  name: tailscale
  namespace: flux-system
spec:
  interval: 1h0m0s
  url: https://pkgs.tailscale.com/helmcharts
EOF
```

### Step 4: Create OAuth Secret (Encrypted)

```bash
# Create plain secret first
cat > /tmp/secret-oauth.yaml << EOF
apiVersion: v1
kind: Secret
metadata:
  name: tailscale-operator-oauth
  namespace: tailscale
type: Opaque
stringData:
  client_id: "<YOUR_OAUTH_CLIENT_ID>"
  client_secret: "<YOUR_OAUTH_CLIENT_SECRET>"
EOF

# Encrypt with SOPS
sops -e /tmp/secret-oauth.yaml > k8s/infrastructure/tailscale/secret-oauth.sops.yaml

# Clean up plain text
rm /tmp/secret-oauth.yaml
```

### Step 5: Create HelmRelease

```bash
cat > k8s/infrastructure/tailscale/helmrelease.yaml << 'EOF'
apiVersion: helm.toolkit.fluxcd.io/v2
kind: HelmRelease
metadata:
  name: tailscale-operator
  namespace: tailscale
spec:
  interval: 10m0s
  chart:
    spec:
      chart: tailscale-operator
      sourceRef:
        kind: HelmRepository
        name: tailscale
        namespace: flux-system
      version: "1.76.1"  # Pin to specific version
  releaseName: tailscale-operator
  install:
    createNamespace: false
    remediation:
      retries: 3
  upgrade:
    remediation:
      retries: 3
  valuesFrom:
    - kind: Secret
      name: tailscale-operator-oauth
      valuesKey: client_id
      targetPath: oauth.clientId
    - kind: Secret
      name: tailscale-operator-oauth
      valuesKey: client_secret
      targetPath: oauth.clientSecret
EOF
```

### Step 6: Create Gateway

```bash
cat > k8s/infrastructure/tailscale/gateway.yaml << 'EOF'
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
EOF
```

### Step 7: Create HTTPRoute for Longhorn

```bash
cat > k8s/infrastructure/tailscale/httproute-longhorn.yaml << 'EOF'
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
    - "longhorn"
  rules:
    - matches:
        - path:
            type: PathPrefix
            value: /
      backendRefs:
        - name: longhorn-frontend
          port: 80
EOF
```

### Step 8: Create Kustomization

```bash
cat > k8s/infrastructure/tailscale/kustomization.yaml << 'EOF'
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - namespace.yaml
  - helmrepository.yaml
  - secret-oauth.sops.yaml
  - helmrelease.yaml
  - gateway.yaml
  - httproute-longhorn.yaml
EOF
```

### Step 9: Update Infrastructure Kustomization

Add `tailscale/` to `k8s/infrastructure/kustomization.yaml`:

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - cilium/
  - longhorn/
  - tailscale/  # Add this line
```

### Step 10: Commit and Push

```bash
git add k8s/infrastructure/tailscale/ k8s/infrastructure/kustomization.yaml
git commit -m "feat: add Tailscale Kubernetes Operator installation with Gateway API"
git push
```

### Step 11: Verify Deployment

```bash
# Wait for Flux to reconcile (or trigger manually)
flux reconcile kustomization infrastructure --with-source

# Check HelmRelease status
kubectl get helmrelease -n tailscale

# Check operator pod
kubectl get pods -n tailscale

# Check operator logs
kubectl logs -n tailscale -l app.kubernetes.io/name=tailscale-operator

# Check Gateway status
kubectl get gateway -n tailscale

# Check HTTPRoute status
kubectl get httproute -n longhorn-system
```

## Verification Checklist

- [ ] Operator Pod is `Running` in `tailscale` namespace
- [ ] HelmRelease shows `Ready=True`
- [ ] Device `tailscale-operator` appears in Tailscale admin console
- [ ] Device has `tag:k8s-operator` tag
- [ ] Gateway shows `Programmed=True`
- [ ] HTTPRoute shows `Accepted=True`
- [ ] Longhorn UI accessible via `https://longhorn.<tailnet>.ts.net`

## Troubleshooting

### Operator Pod Not Starting

```bash
# Check events
kubectl describe pod -n tailscale -l app.kubernetes.io/name=tailscale-operator

# Check HelmRelease status
kubectl describe helmrelease tailscale-operator -n tailscale
```

### OAuth Authentication Failed

```bash
# Verify secret exists and has correct keys
kubectl get secret tailscale-operator-oauth -n tailscale -o yaml

# Check operator logs for auth errors
kubectl logs -n tailscale -l app.kubernetes.io/name=tailscale-operator | grep -i auth
```

### Gateway/HTTPRoute Not Working

```bash
# Check Gateway status
kubectl describe gateway tailscale-gateway -n tailscale

# Check HTTPRoute status
kubectl describe httproute longhorn-ui -n longhorn-system

# Check for proxy StatefulSet
kubectl get statefulset -n tailscale

# Check proxy logs
kubectl logs -n tailscale -l tailscale.com/parent-resource=longhorn-ui
```

## Next Steps

After successful deployment:

1. **Configure ACLs** in Tailscale admin console to control access to Longhorn UI
2. **Add more HTTPRoutes** for other services you want to expose (referencing the same Gateway)
3. **Set up MagicDNS** for easier hostname resolution
4. **Enable HTTPS** (automatic with Tailscale certificates)
