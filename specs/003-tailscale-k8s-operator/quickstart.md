# Quickstart: Tailscale Kubernetes Operator

**Feature**: 003-tailscale-k8s-operator
**Date**: 2026-01-03

## Prerequisites

1. **Tailscale Account**: Active Tailscale account with admin access
2. **Cluster Access**: kubectl configured for the homelab cluster
3. **SOPS Key**: Age key available for secret encryption
4. **Flux**: Flux CD running and reconciling

## Step 1: Create OAuth Client in Tailscale

1. Go to [Tailscale Admin Console](https://login.tailscale.com/admin/settings/oauth)
2. Click **Generate OAuth Client**
3. Configure:
   - **Description**: `homelab-k8s-operator`
   - **Scopes**:
     - `Devices Core` (write)
     - `Auth Keys` (write)
     - `Services` (write)
   - **Tags**: `tag:k8s-operator`, `tag:k8s`
4. Save the **Client ID** and **Client Secret**

## Step 2: Create Encrypted Secret

```bash
# Navigate to tailscale directory
cd k8s/infrastructure/tailscale

# Create secret file (will be encrypted)
cat > secret-oauth-credentials.yaml << EOF
apiVersion: v1
kind: Secret
metadata:
  name: tailscale-operator-oauth
  namespace: tailscale
type: Opaque
stringData:
  client_id: "<YOUR_CLIENT_ID>"
  client_secret: "<YOUR_CLIENT_SECRET>"
EOF

# Encrypt with SOPS
sops -e -i secret-oauth-credentials.yaml
mv secret-oauth-credentials.yaml secret-oauth-credentials.sops.yaml
```

## Step 3: Apply Configuration

```bash
# Commit and push changes
git add k8s/infrastructure/tailscale/
git commit -m "feat: add Tailscale Operator configuration"
git push

# Force reconciliation (optional)
flux reconcile kustomization infrastructure --with-source
```

## Step 4: Verify Deployment

```bash
# Check operator pod
kubectl get pods -n tailscale

# Check operator logs
kubectl logs -n tailscale -l app.kubernetes.io/name=tailscale-operator

# Verify in Tailscale admin console
# - Device "tailscale-operator" should appear with tag:k8s-operator
```

## Step 5: Expose Longhorn UI (Smoke Test)

```bash
# Apply the smoke test Ingress
kubectl apply -f k8s/infrastructure/tailscale/smoke/

# Check Ingress status
kubectl get ingress -n longhorn-system longhorn-ui

# Access from tailnet device
curl -I https://longhorn.<your-tailnet>.ts.net
```

## Verification Checklist

- [ ] Operator pod is Running
- [ ] Operator appears in Tailscale admin console
- [ ] ProxyGroup status is Ready (if using)
- [ ] Longhorn UI accessible via Tailscale hostname
- [ ] TLS certificate is valid

## Troubleshooting

### Operator pod in CrashLoopBackOff

```bash
# Check logs for OAuth errors
kubectl logs -n tailscale -l app.kubernetes.io/name=tailscale-operator

# Common causes:
# - Invalid OAuth credentials
# - Missing scopes on OAuth client
# - Network connectivity issues
```

### Ingress not accessible

```bash
# Check Ingress events
kubectl describe ingress -n longhorn-system longhorn-ui

# Check proxy pod logs
kubectl logs -n tailscale -l tailscale.com/parent-resource-type=ingress

# Verify Cilium compatibility
kubectl get ciliumnetworkpolicies -A
```

### ProxyGroup not ready

```bash
# Check ProxyGroup status
kubectl describe proxygroup -n tailscale ingress-proxies

# Check StatefulSet
kubectl get statefulset -n tailscale
```

## Clean Up (if needed)

```bash
# Remove Ingress
kubectl delete ingress -n longhorn-system longhorn-ui

# Remove operator (via Git revert)
git revert HEAD
git push
flux reconcile kustomization infrastructure
```

## Next Steps

1. Configure ACLs in Tailscale admin console
2. Add more services to expose via Ingress
3. Set up monitoring for proxy health
