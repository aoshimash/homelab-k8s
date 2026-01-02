# Contract: Gateway API CRD Reconciliation

**Feature**: 004-gateway-api
**Date**: 2026-01-03

## Overview

This contract defines the expected behavior of Flux reconciliation for Gateway API CRD installation.

## Flux GitRepository Contract

### Input

| Field | Value | Description |
|-------|-------|-------------|
| url | `https://github.com/kubernetes-sigs/gateway-api` | Upstream repository |
| ref.tag | `v1.4.1` | Pinned version tag |
| interval | `1h` | Sync interval |

### Expected Output

| Condition | Status | Reason |
|-----------|--------|--------|
| Ready | True | Artifact fetched successfully |
| ArtifactInStorage | True | Tag v1.4.1 content cached |

### Failure Scenarios

| Scenario | Expected Behavior |
|----------|-------------------|
| Network unavailable | Retry with exponential backoff, use cached artifact if available |
| Tag not found | Report error, Kustomization will not reconcile |
| Repository unavailable | Retry, alert if persistent |

## Flux Kustomization Contract

### Input

| Field | Value | Description |
|-------|-------|-------------|
| sourceRef | GitRepository/gateway-api | Source reference |
| path | `./config/crd/standard` | CRD directory path |
| prune | `false` | Do not delete CRDs on removal |
| interval | `10m0s` | Reconciliation interval |

### Expected Output

| Condition | Status | Reason |
|-----------|--------|--------|
| Ready | True | Applied revision: v1.4.1/sha |
| Healthy | True | All resources healthy |

### Applied Resources

The Kustomization should apply the following CRDs:

| CRD Name | API Group | Expected |
|----------|-----------|----------|
| gatewayclasses.gateway.networking.k8s.io | gateway.networking.k8s.io | ✓ |
| gateways.gateway.networking.k8s.io | gateway.networking.k8s.io | ✓ |
| httproutes.gateway.networking.k8s.io | gateway.networking.k8s.io | ✓ |
| referencegrants.gateway.networking.k8s.io | gateway.networking.k8s.io | ✓ |
| grpcroutes.gateway.networking.k8s.io | gateway.networking.k8s.io | ✓ |

### Failure Scenarios

| Scenario | Expected Behavior |
|----------|-------------------|
| Source not ready | Wait for GitRepository to become ready |
| Invalid YAML | Report error, do not apply partial changes |
| CRD conflict | Report error if existing CRD has incompatible schema |
| Permission denied | Report RBAC error |

## CRD Installation Contract

### Pre-conditions

1. Flux is installed and operational in `flux-system` namespace
2. Network connectivity to GitHub is available
3. Cluster has sufficient permissions to create CRDs

### Post-conditions

1. All Gateway API CRDs are registered in the cluster
2. CRDs are in `Established` condition
3. Existing Gateway/HTTPRoute resources (if any) become valid
4. No API errors for `gateway.networking.k8s.io` resources

### Verification Commands

```bash
# Verify CRDs are installed
kubectl get crd | grep gateway.networking.k8s.io

# Expected output:
# gatewayclasses.gateway.networking.k8s.io     <timestamp>
# gateways.gateway.networking.k8s.io           <timestamp>
# grpcroutes.gateway.networking.k8s.io         <timestamp>
# httproutes.gateway.networking.k8s.io         <timestamp>
# referencegrants.gateway.networking.k8s.io    <timestamp>

# Verify Flux resources
kubectl get gitrepository gateway-api -n flux-system
kubectl get kustomization gateway-api-crds -n flux-system

# Verify CRD status
kubectl get crd gateways.gateway.networking.k8s.io -o jsonpath='{.status.conditions[?(@.type=="Established")].status}'
# Expected: True
```

## Integration with Tailscale Operator

### Pre-conditions

1. Gateway API CRDs are installed and established
2. Tailscale Operator is running

### Expected Behavior

| Resource | Namespace | Expected Status |
|----------|-----------|-----------------|
| Gateway `tailscale-gateway` | tailscale | Programmed=True |
| HTTPRoute `longhorn-ui` | longhorn-system | Accepted=True |

### Verification Commands

```bash
# Verify Gateway status
kubectl get gateway tailscale-gateway -n tailscale -o jsonpath='{.status.conditions[?(@.type=="Programmed")].status}'
# Expected: True

# Verify HTTPRoute status
kubectl get httproute longhorn-ui -n longhorn-system -o jsonpath='{.status.parents[0].conditions[?(@.type=="Accepted")].status}'
# Expected: True

# Verify Tailscale device created
# Check Tailscale admin console for new device
```

## SLA Expectations

| Metric | Target |
|--------|--------|
| CRD installation time | < 2 minutes from Flux reconciliation |
| Gateway programmed time | < 5 minutes from CRD installation |
| Reconciliation interval | 10 minutes |
| Recovery from failure | < 15 minutes with automatic retry |
