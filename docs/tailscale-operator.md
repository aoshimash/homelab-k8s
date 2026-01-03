# Tailscale Kubernetes Operator Setup

This document describes the setup and configuration of the Tailscale Kubernetes Operator for exposing cluster services to the tailnet.

## Overview

The Tailscale Kubernetes Operator allows you to:
- Expose Kubernetes services to your tailnet via Ingress
- Use ProxyGroup for resource-efficient shared proxy infrastructure
- Automatic TLS certificate provisioning via Tailscale HTTPS

## Prerequisites

### Tailscale Admin Console Configuration

Before deploying the operator, you must configure the following in the [Tailscale Admin Console](https://login.tailscale.com/admin):

#### 1. Create ACL Tags

Add the following tags to your ACL policy under **Access controls** → `tagOwners`:

```json
{
  "tagOwners": {
    "tag:k8s-operator": [],
    "tag:k8s": ["tag:k8s-operator"]
  }
}
```

- `tag:k8s-operator`: Tag for the operator itself (empty array means only the tailnet admin can assign it)
- `tag:k8s`: Tag for proxy devices, owned by `tag:k8s-operator`

#### 2. Create OAuth Client

1. Go to **Settings** → **OAuth clients**
2. Click **Generate OAuth client**
3. Configure the following scopes:
   - **Devices** - Read & Write
   - **Auth Keys** - Read & Write  
   - **Services** - Read & Write (required for Ingress with ProxyGroup)
4. Set tags: `tag:k8s-operator`, `tag:k8s`
5. Save the `client_id` and `client_secret`

> ⚠️ **Important**: The `Services` scope is required for creating VIP Services used by ProxyGroup-based Ingress. Without this scope, the operator will fail with `401 Unauthorized` when creating Ingress resources.

#### 3. Configure Auto-Approvers for Services

To automatically approve VIP Services created by the operator, add the following to your ACL policy:

```json
{
  "autoApprovers": {
    "services": {
      "svc:*": ["tag:k8s-operator"]
    }
  }
}
```

This allows devices tagged with `tag:k8s-operator` to automatically advertise services without manual approval.

> **Note**: Without this configuration, each new Ingress will create a Service in "Needs approval" state, requiring manual approval in the Tailscale Admin Console under **Services**.

### ACL Policy Changes Summary

Add the following to your existing ACL policy:

```json
{
  "tagOwners": {
    "tag:k8s-operator": [],
    "tag:k8s": ["tag:k8s-operator"]
  },
  "autoApprovers": {
    "services": {
      "svc:*": ["tag:k8s-operator"]
    }
  }
}
```

## Kubernetes Resources

### Directory Structure

```
k8s/infrastructure/
├── tailscale/                    # Base operator resources
│   ├── namespace.yaml
│   ├── helmrepository.yaml
│   ├── helmrelease.yaml
│   ├── secret-oauth-credentials.sops.yaml
│   └── kustomization.yaml
└── tailscale-config/             # CRD-dependent resources
    ├── proxygroup.yaml
    ├── kustomization.yaml
    └── smoke/
        ├── ingress-longhorn.yaml
        └── kustomization.yaml
```

### Creating OAuth Secret

Create the SOPS-encrypted secret with your OAuth credentials:

```bash
# Create the secret file
cat <<EOF > k8s/infrastructure/tailscale/secret-oauth-credentials.yaml
apiVersion: v1
kind: Secret
metadata:
  name: tailscale-operator-oauth
  namespace: tailscale
type: Opaque
stringData:
  client_id: "your-client-id"
  client_secret: "tskey-client-your-client-secret"
EOF

# Encrypt with SOPS
sops --encrypt --in-place k8s/infrastructure/tailscale/secret-oauth-credentials.yaml

# Rename to .sops.yaml
mv k8s/infrastructure/tailscale/secret-oauth-credentials.yaml \
   k8s/infrastructure/tailscale/secret-oauth-credentials.sops.yaml
```

### Namespace Configuration

The tailscale namespace requires PodSecurity labels to allow privileged pods:

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

> **Note**: Tailscale proxy pods require privileged mode for network capabilities (sysctl, NET_ADMIN).

## Cilium Compatibility

When running Cilium in kube-proxy replacement mode, there are two important configurations needed to ensure Tailscale proxies work correctly.

### Socket LB Bypass

Configure `socketLB.hostNamespaceOnly: true` to ensure Tailscale proxies can correctly route traffic:

```yaml
# In Cilium HelmRelease values
socketLB:
  hostNamespaceOnly: true
```

This is required because when Cilium runs in kube-proxy replacement mode with socket load balancing enabled in Pods' namespaces, connections from Pods to ClusterIPs go over a TCP socket (instead of going out via Pods' veth devices) and thus bypass Tailscale firewall rules.

### MTU Configuration (Critical for Performance)

#### Background: What is MTU?

MTU (Maximum Transmission Unit) is the maximum packet size that can be sent over a network in a single transmission. When packets exceed the MTU, they must be fragmented, which causes significant performance degradation due to:
- Additional header overhead for each fragment
- Reassembly processing at the receiver
- Entire packet retransmission if any fragment is lost
- TCP performance degradation from window size issues

#### The Problem

**Without Tailscale**, Cilium auto-detects MTU from the physical network interface:

```
Physical NIC (enp1s0f1): MTU 1500
Cilium MTU: 1500 - 50 (VXLAN overhead) = 1450 ✅
```

**After installing Tailscale** (e.g., via Talos extension), a `tailscale0` interface is created with MTU 1280:

```
Physical NIC (enp1s0f1): MTU 1500
tailscale0:              MTU 1280 ← Added by Tailscale

Cilium MTU: min(1500, 1280) = 1280 ❌ (unintentionally lowered)
```

This causes **all cluster-internal communication** to be limited to MTU 1280, not just Tailscale traffic.

#### Why This Causes Severe Performance Issues

The issue is not that MTU 1280 is "slow", but the **MTU mismatch** between what applications expect and what the network allows:

```
Application: "Sending 1450 byte packet"
Cilium network: "Only 1280 allowed"
Result: Packet drop, fragmentation, TCP retransmission timeouts
```

When traffic flows through Tailscale Ingress:

```
audiobookshelf Pod ──[Cilium MTU 1280]──> ProxyGroup Pod ──[Tailscale MTU 1280]──> iPhone

With the fix:
audiobookshelf Pod ──[Cilium MTU 1450]──> ProxyGroup Pod ──[Tailscale MTU 1280]──> iPhone
                     ↑ Fixed!              ↑ Tailscale handles conversion properly
```

The ProxyGroup Pod properly converts between Cilium (1450) and Tailscale (1280) networks. The problem was that Cilium was unnecessarily restricted to 1280.

**Symptoms**:
- Download speeds are significantly slower than expected (10-20x slower)
- Upload speeds may be relatively normal (asymmetric performance)
- `cilium-dbg status --verbose` shows "MTU updated (1280)"

**Diagnosis**:
```bash
# Check current MTU in Cilium
kubectl exec -n kube-system ds/cilium -- cilium-dbg status --verbose | grep -i mtu

# Check interface MTUs
kubectl exec -n kube-system ds/cilium -- ip link show tailscale0 | grep mtu
kubectl exec -n kube-system ds/cilium -- ip link show enp1s0f1 | grep mtu  # physical interface
```

**Solution**: Explicitly set the MTU in Cilium configuration to restore the correct value and prevent Cilium from using `tailscale0`'s low MTU:

```yaml
# In Cilium HelmRelease values
# Note: Cilium Helm chart uses uppercase 'MTU' key
MTU: 1450  # Use physical interface MTU minus overhead (1500 - 50 for VXLAN)
```

> **Note**: The MTU value of 1450 accounts for VXLAN encapsulation overhead. If using a different tunnel mode, adjust accordingly.
> **Important**: The Cilium Helm chart uses uppercase `MTU` (not lowercase `mtu`).

### Complete Cilium Configuration for Tailscale

```yaml
# In Cilium HelmRelease values
socketLB:
  hostNamespaceOnly: true
MTU: 1450
```

After applying these changes, restart Cilium pods:

```bash
kubectl rollout restart daemonset/cilium -n kube-system
```

### Verification

After applying the MTU fix, verify the configuration:

```bash
# Check Cilium interface MTU (should show 1450)
kubectl exec -n kube-system ds/cilium -- ip link show cilium_host | grep mtu

# Check Cilium status (should NOT show "MTU updated (1280)")
kubectl exec -n kube-system ds/cilium -- cilium-dbg status --verbose | grep -i mtu
```

### Actual Results (2026-01-03)

After applying the MTU fix in our homelab cluster:

| Metric | Before (MTU 1280) | After (MTU 1450) | Improvement |
|--------|-------------------|------------------|-------------|
| Tailscale Ingress response time | 459ms | 23ms | **~20x faster** |
| port-forward response time | 13ms | 13ms | (unchanged) |
| `cilium_host` MTU | 1280 | 1450 | ✅ Fixed |

The fix resolved the slow download issue experienced when accessing audiobookshelf via Tailscale Ingress from iPhone.

### References

- [Tailscale KB: Cilium in kube-proxy replacement mode](https://tailscale.com/kb/1236/kubernetes-operator#cilium-in-kube-proxy-replacement-mode)

## Creating an Ingress

### Ingress with ProxyGroup

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: my-service
  namespace: my-namespace
  annotations:
    tailscale.com/proxy-group: ingress-proxies
spec:
  ingressClassName: tailscale
  defaultBackend:
    service:
      name: my-service
      port:
        number: 80
  tls:
    - hosts:
        - my-service
```

The service will be accessible at `https://my-service.<tailnet>.ts.net`.

### Important Notes

- Use `defaultBackend` instead of `rules` for backend configuration
- Use `tls.hosts` for hostname specification
- The `tailscale.com/proxy-group` annotation references the ProxyGroup

## Verification

### Check Operator Status

```bash
kubectl get pods -n tailscale
kubectl logs -n tailscale deployment/operator --tail=50
```

### Check ProxyGroup Status

```bash
kubectl get proxygroup -n tailscale
kubectl get statefulset -n tailscale
```

### Check Ingress Status

```bash
kubectl get ingress -n <namespace>
kubectl describe ingress <name> -n <namespace>
```

### Test Connectivity

```bash
# From a device on the tailnet
curl https://my-service.<tailnet>.ts.net
```

## Troubleshooting

### OAuth Token Invalid (401 Unauthorized)

**Symptom**: Operator logs show `oauth2: cannot fetch token: 401 Unauthorized`

**Causes**:
1. OAuth client credentials are incorrect
2. OAuth client was regenerated (new secret)
3. OAuth client lacks required scopes (especially `Services`)

**Solution**:
1. Verify credentials in Tailscale Admin Console
2. Update the Kubernetes secret if credentials changed
3. Restart the operator: `kubectl rollout restart deployment/operator -n tailscale`

### Service Needs Approval

**Symptom**: Ingress shows no ADDRESS, Service in Admin Console shows "Needs approval"

**Solution**:
1. Manually approve in Tailscale Admin Console → Services
2. Add `autoApprovers` to ACL policy for automatic approval (see [Configure Auto-Approvers](#3-configure-auto-approvers-for-services))

### ProxyGroup Pod Fails to Start

**Symptom**: ProxyGroup pod in Error or CrashLoopBackOff

**Check**:
```bash
kubectl describe pod -n tailscale ingress-proxies-0
kubectl logs -n tailscale ingress-proxies-0
```

**Common causes**:
1. PodSecurity policy blocking privileged pods - add labels to namespace
2. Cilium socketLB configuration - enable `hostNamespaceOnly`
3. Network connectivity issues - check Cilium status

### Ingress Shows "No Valid Backends"

**Symptom**: `InvalidIngressBackend` or `NoValidBackends` warnings

**Solution**: Use the correct Ingress format with `defaultBackend` and `tls.hosts` instead of `rules.host`.

### Slow Download Speeds via Tailscale Ingress

**Symptom**: 
- Downloads are extremely slow (10-50x slower than expected)
- Uploads are relatively normal
- Direct ping latency is low (e.g., 3ms for LAN connections)
- `kubectl port-forward` shows normal speeds

**Diagnosis**:
```bash
# Compare Tailscale Ingress vs direct port-forward
# Via Tailscale (slow)
curl -o /dev/null -w "time: %{time_total}s\n" https://myservice.tailnet.ts.net/healthcheck

# Via port-forward (should be faster)
kubectl port-forward svc/myservice 8080:80 &
curl -o /dev/null -w "time: %{time_total}s\n" http://localhost:8080/healthcheck

# Check Cilium MTU
kubectl exec -n kube-system ds/cilium -- cilium-dbg status --verbose | grep -i mtu
```

**Cause**: Cilium is using `tailscale0` interface's MTU (1280) instead of the physical interface MTU (1500).

**Solution**: See [MTU Configuration](#mtu-configuration-critical-for-performance) in the Cilium Compatibility section.

## References

- [Tailscale Kubernetes Operator Documentation](https://tailscale.com/kb/1236/kubernetes-operator)
- [ProxyGroup Configuration](https://tailscale.com/kb/1236/kubernetes-operator#optional-pre-creating-a-proxygroup)
- [Cilium Compatibility](https://tailscale.com/kb/1236/kubernetes-operator#cilium-in-kube-proxy-replacement-mode)
- [ACL Auto-Approvers](https://tailscale.com/kb/1541/kubernetes-operator-multi-cluster-ingress)
