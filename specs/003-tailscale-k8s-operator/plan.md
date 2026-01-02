# Implementation Plan: Tailscale Kubernetes Operator

**Branch**: `003-tailscale-k8s-operator` | **Date**: 2026-01-03 | **Spec**: [spec.md](./spec.md)
**Input**: Feature specification from `/specs/003-tailscale-k8s-operator/spec.md`

## Summary

Deploy Tailscale Kubernetes Operator via GitOps (Flux) to expose cluster services to the tailnet using Tailscale Ingress. The implementation includes Cilium kube-proxy replacement compatibility, ProxyGroup for resource efficiency, and SOPS-encrypted OAuth credentials.

## Technical Context

**Language/Version**: YAML (Kubernetes manifests), Helm charts
**Primary Dependencies**: Flux CD, Tailscale Operator Helm chart, Cilium CNI
**Storage**: N/A (stateless operator)
**Testing**: Manual verification via tailnet access, kubectl status checks
**Target Platform**: Kubernetes (Talos Linux single-node cluster)
**Project Type**: GitOps infrastructure configuration
**Performance Goals**: Service accessible within 5 minutes of Ingress creation
**Constraints**: Single-node cluster, Cilium with kubeProxyReplacement: true
**Scale/Scope**: Single operator, 1 ProxyGroup replica, initial smoke test with Longhorn UI

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-check after Phase 1 design.*

| Principle | Status | Notes |
|-----------|--------|-------|
| I. GitOps-First | ✅ PASS | All resources deployed via Flux HelmRelease/Kustomization |
| II. Declarative Configuration | ✅ PASS | YAML manifests, Helm values, no imperative commands |
| III. Secrets Management | ✅ PASS | OAuth credentials encrypted with SOPS+age |
| IV. Immutable Infrastructure | ✅ PASS | No node-level changes required |
| V. Documentation as Code | ✅ PASS | Quickstart and operational docs included |

**Post-Phase 1 Re-check**: All gates pass. No violations.

## Project Structure

### Documentation (this feature)

```text
specs/003-tailscale-k8s-operator/
├── plan.md              # This file
├── spec.md              # Feature specification
├── research.md          # Phase 0 output
├── data-model.md        # Phase 1 output
├── quickstart.md        # Phase 1 output
├── contracts/
│   └── reconciliation.md # Flux reconciliation contract
├── checklists/
│   └── requirements.md  # Spec quality checklist
└── tasks.md             # Phase 2 output (created by /speckit.tasks)
```

### Source Code (repository root)

```text
k8s/infrastructure/tailscale/
├── kustomization.yaml              # Kustomize entry point
├── namespace.yaml                  # tailscale namespace
├── helmrepository.yaml             # Flux HelmRepository for Tailscale charts
├── helmrelease.yaml                # Tailscale Operator HelmRelease
├── secret-oauth-credentials.sops.yaml  # SOPS-encrypted OAuth secret
├── proxygroup.yaml                 # ProxyGroup for shared ingress proxies
└── smoke/
    ├── kustomization.yaml          # Smoke test resources
    └── ingress-longhorn.yaml       # Longhorn UI Ingress for testing

k8s/infrastructure/cilium/
└── helmrelease.yaml                # Updated with socketLB.hostNamespaceOnly
```

**Structure Decision**: Follow existing infrastructure pattern (`k8s/infrastructure/<component>/`) with Kustomize + Helm. Smoke test resources in subdirectory, disabled by default.

## Key Design Decisions

### 1. Cilium Compatibility

**Problem**: Cilium in kube-proxy replacement mode intercepts socket-level traffic, bypassing Tailscale firewall rules.

**Solution**: Configure `socketLB.hostNamespaceOnly: true` in Cilium HelmRelease values.

**Reference**: [Tailscale KB - Cilium Compatibility](https://tailscale.com/kb/1236/kubernetes-operator#cilium-in-kube-proxy-replacement-mode)

### 2. ProxyGroup vs Per-Service Proxies

**Problem**: Default behavior creates one proxy pod per Ingress, increasing resource consumption.

**Solution**: Use ProxyGroup with 1 replica for single-node cluster. Multiple Ingress resources share the same proxy infrastructure.

**Trade-off**: Single replica means no HA during proxy pod restarts. Acceptable for homelab.

### 3. OAuth Credential Injection

**Problem**: HelmRelease needs OAuth credentials without exposing them in Git.

**Solution**: Create SOPS-encrypted Secret, reference via `valuesFrom` in HelmRelease.

```yaml
spec:
  valuesFrom:
    - kind: Secret
      name: tailscale-operator-oauth
      valuesKey: client_id
      targetPath: oauth.clientId
    - kind: Secret
      name: tailscale-operator-oauth
      valuesKey: client_secret
      targetPath: oauth.clientSecret
```

### 4. ACL Tag Strategy

**Decision**:
- Operator: `tag:k8s-operator` (management operations)
- Proxies: `tag:k8s` (service access)

**Rationale**: Simple, uniform access control. Can be refined later if needed.

## Dependencies

| Dependency | Version | Purpose |
|------------|---------|---------|
| Tailscale Operator Helm Chart | Latest stable | Operator deployment |
| Flux CD | Existing | GitOps reconciliation |
| Cilium | 1.18.5 (existing) | CNI with kube-proxy replacement |
| SOPS | Existing | Secret encryption |
| Longhorn | Existing | Smoke test target service |

## Risk Assessment

| Risk | Likelihood | Impact | Mitigation |
|------|------------|--------|------------|
| OAuth token expiry | Low | Medium | Monitor operator logs, document renewal |
| Cilium config breaks existing services | Low | High | Test in isolation, rollback plan ready |
| ProxyGroup not supported in chart version | Low | Medium | Fall back to per-service proxies |
| Tailnet ACL blocks access | Medium | Low | Document ACL requirements in quickstart |

## Complexity Tracking

No constitution violations requiring justification.

## Phase Outputs

- [x] **Phase 0**: [research.md](./research.md) - All technical decisions documented
- [x] **Phase 1**: [data-model.md](./data-model.md) - Kubernetes resource definitions
- [x] **Phase 1**: [contracts/reconciliation.md](./contracts/reconciliation.md) - Flux reconciliation contract
- [x] **Phase 1**: [quickstart.md](./quickstart.md) - Deployment guide
- [x] **Phase 2**: [tasks.md](./tasks.md) - Task breakdown
