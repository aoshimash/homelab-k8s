# Contract: Flux Reconciliation for Tailscale Operator

**Feature**: 003-tailscale-k8s-operator
**Date**: 2026-01-02

## Overview

This contract defines the expected behavior of Flux CD when reconciling Tailscale Operator resources.

## Reconciliation Flow

### 1. HelmRepository Sync

**Trigger**: Interval (1h) or manual `flux reconcile source helm tailscale`

**Input**:
- Repository URL: `https://pkgs.tailscale.com/helmcharts`

**Expected Output**:
```yaml
status:
  conditions:
    - type: Ready
      status: "True"
      reason: Succeeded
  artifact:
    revision: <chart-index-checksum>
```

**Failure Modes**:
| Condition | Reason | Recovery |
|-----------|--------|----------|
| Network unreachable | FetchFailed | Retry on next interval |
| Invalid URL | URLInvalid | Fix HelmRepository spec |

### 2. HelmRelease Reconciliation

**Trigger**: Interval (10m), source update, or manual `flux reconcile helmrelease tailscale-operator -n tailscale`

**Input**:
- Chart: `tailscale-operator`
- Version: `<pinned-version>`
- Values: OAuth credentials from Secret

**Expected Output**:
```yaml
status:
  conditions:
    - type: Ready
      status: "True"
      reason: InstallSucceeded  # or UpgradeSucceeded
  lastAppliedRevision: <chart-version>
  lastAttemptedRevision: <chart-version>
```

**Failure Modes**:
| Condition | Reason | Recovery |
|-----------|--------|----------|
| Chart not found | ChartPullFailed | Check version exists |
| Invalid values | InstallFailed | Fix HelmRelease values |
| Secret missing | DependencyNotReady | Create/fix Secret |
| Pod crash loop | InstallFailed | Check operator logs |

### 3. Secret Decryption (SOPS)

**Trigger**: Kustomization reconciliation

**Input**:
- Encrypted file: `secret-oauth.sops.yaml`
- Decryption key: `sops-age` Secret in flux-system

**Expected Output**:
- Kubernetes Secret created in `tailscale` namespace
- Decrypted values available to HelmRelease

**Failure Modes**:
| Condition | Reason | Recovery |
|-----------|--------|----------|
| Decryption failed | DecryptionFailed | Check age key |
| Invalid YAML | BuildFailed | Fix encrypted file |

### 4. Ingress Processing (Tailscale Operator)

**Trigger**: Ingress resource created/updated

**Input**:
- Ingress with `ingressClassName: tailscale`
- Backend service reference

**Expected Output**:
- StatefulSet created in `tailscale` namespace
- Device registered in Tailscale admin console
- Ingress status updated with hostname

```yaml
status:
  loadBalancer:
    ingress:
      - hostname: longhorn.<tailnet>.ts.net
```

**Failure Modes**:
| Condition | Reason | Recovery |
|-----------|--------|----------|
| OAuth invalid | AuthenticationFailed | Fix credentials |
| Backend not found | BackendNotFound | Check service exists |
| Tailscale API error | APIError | Check Tailscale status |

## Verification Commands

### Check HelmRepository Status
```bash
kubectl get helmrepository tailscale -n flux-system -o yaml
flux get source helm tailscale
```

### Check HelmRelease Status
```bash
kubectl get helmrelease tailscale-operator -n tailscale -o yaml
flux get helmrelease tailscale-operator -n tailscale
```

### Check Operator Pod
```bash
kubectl get pods -n tailscale -l app.kubernetes.io/name=tailscale-operator
kubectl logs -n tailscale -l app.kubernetes.io/name=tailscale-operator
```

### Check Ingress Status
```bash
kubectl get ingress longhorn-ui -n longhorn-system -o yaml
```

### Check Tailscale Proxy
```bash
kubectl get statefulset -n tailscale
kubectl get pods -n tailscale
```

## Success Criteria Mapping

| Success Criteria | Verification |
|------------------|--------------|
| SC-001: Pod Running in 5 min | `kubectl get pods -n tailscale` shows Running |
| SC-002: Device in admin console | Check Tailscale admin console |
| SC-003: HelmRelease Ready | `flux get hr tailscale-operator -n tailscale` shows Ready |
| SC-004: Credentials encrypted | `cat secret-oauth.sops.yaml` shows ENC[] |
| SC-005: Upgradeable | Update version, verify reconciliation |
| SC-006: Longhorn UI accessible | Access `https://longhorn.<tailnet>.ts.net` |

## Dependency Order

```
1. Namespace (tailscale)
   └── 2. Secret (tailscale-operator-oauth)
       └── 3. HelmRelease (tailscale-operator)
           └── 4. Ingress (longhorn-ui) [in longhorn-system]
               └── 5. Tailscale Proxy (auto-created)
```

**Note**: HelmRepository is in `flux-system` namespace and can be created independently.
