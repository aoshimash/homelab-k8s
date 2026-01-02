# Contract: Flux Reconciliation for Tailscale Operator

**Feature**: 003-tailscale-k8s-operator
**Date**: 2026-01-03

## Overview

This contract defines the expected behavior of Flux reconciliation for the Tailscale Kubernetes Operator deployment.

## Reconciliation Flow

### 1. HelmRepository Sync

**Trigger**: Interval (1h) or manual refresh

**Input**:
- Repository URL: `https://pkgs.tailscale.com/helmcharts`

**Expected Output**:
- Artifact available with chart index
- Status: `Ready=True`

**Failure Modes**:
| Condition | Status | Action |
|-----------|--------|--------|
| Network unreachable | Ready=False | Retry on next interval |
| Invalid URL | Ready=False | Manual fix required |

### 2. Secret Decryption

**Trigger**: Kustomization reconciliation

**Input**:
- Encrypted file: `secret-oauth-credentials.sops.yaml`
- Decryption key: `sops-age` secret in `flux-system`

**Expected Output**:
- Decrypted Secret in `tailscale` namespace
- Fields: `client_id`, `client_secret`

**Failure Modes**:
| Condition | Status | Action |
|-----------|--------|--------|
| Missing age key | Reconciliation failed | Verify sops-age secret |
| Invalid SOPS file | Reconciliation failed | Re-encrypt with correct key |

### 3. HelmRelease Installation

**Trigger**: HelmRepository artifact ready + Secret available

**Input**:
- Chart: `tailscale-operator`
- Version: pinned
- Values: OAuth credentials from secret

**Expected Output**:
- Operator Deployment in `tailscale` namespace
- ServiceAccount with RBAC
- CRDs installed (ProxyGroup, etc.)
- Status: `Ready=True`

**Failure Modes**:
| Condition | Status | Action |
|-----------|--------|--------|
| Invalid OAuth credentials | Pods CrashLoopBackOff | Verify credentials in Tailscale console |
| CRD conflict | Install failed | Check existing Tailscale resources |
| Resource limits | Pending pods | Adjust resource requests |

### 4. Operator Registration

**Trigger**: Operator pod running

**Input**:
- OAuth client ID/secret
- Hostname: `tailscale-operator`

**Expected Output**:
- Device appears in Tailscale admin console
- Tag: `tag:k8s-operator`
- Status: Connected

**Verification**:
```bash
# Check operator pod status
kubectl get pods -n tailscale -l app.kubernetes.io/name=tailscale-operator

# Check operator logs for successful registration
kubectl logs -n tailscale -l app.kubernetes.io/name=tailscale-operator | grep -i "registered"
```

### 5. ProxyGroup Creation

**Trigger**: ProxyGroup resource applied

**Input**:
- ProxyGroup manifest with `type: ingress`, `replicas: 1`

**Expected Output**:
- StatefulSet created with 1 replica
- Proxy pod registered in tailnet
- ProxyGroup status: `ProxyGroupReady=True`

**Verification**:
```bash
# Check ProxyGroup status
kubectl get proxygroup -n tailscale

# Wait for ready
kubectl wait proxygroup ingress-proxies --for=condition=ProxyGroupReady=true -n tailscale
```

### 6. Ingress Provisioning

**Trigger**: Ingress with `ingressClassName: tailscale` created

**Input**:
- Ingress resource in target namespace
- Backend service reference

**Expected Output**:
- Proxy pod (or ProxyGroup) handles traffic
- TLS certificate provisioned
- Hostname accessible in tailnet

**Verification**:
```bash
# Check Ingress status
kubectl get ingress -n longhorn-system longhorn-ui

# Verify from tailnet device
curl -I https://longhorn.<tailnet>.ts.net
```

## Success Criteria Mapping

| Success Criteria | Reconciliation Step | Verification |
|-----------------|---------------------|--------------|
| SC-001 | Ingress Provisioning | Access from tailnet device |
| SC-002 | Ingress Provisioning | HTTPS connection succeeds |
| SC-003 | Secret Decryption | No plaintext in Git |
| SC-005 | Operator Registration | Device in admin console |
| SC-006 | ProxyGroup Creation | Fixed pod count |
| SC-007 | All steps | Traffic routing works |

## Rollback Procedure

If reconciliation fails:

1. **Check Flux status**:
   ```bash
   flux get all -n flux-system
   ```

2. **Suspend reconciliation**:
   ```bash
   flux suspend kustomization infrastructure
   ```

3. **Revert Git commit**:
   ```bash
   git revert HEAD
   git push
   ```

4. **Resume reconciliation**:
   ```bash
   flux resume kustomization infrastructure
   ```

## Dependencies

| Resource | Depends On |
|----------|------------|
| HelmRelease | HelmRepository, OAuth Secret |
| ProxyGroup | Operator running |
| Ingress | ProxyGroup (optional), Backend service |
