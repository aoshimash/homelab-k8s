# Implementation Plan: Gateway API CRD Installation

**Branch**: `004-gateway-api` | **Date**: 2026-01-03 | **Spec**: [spec.md](./spec.md)
**Input**: Feature specification from `/specs/004-gateway-api/spec.md`

## Summary

Install Kubernetes Gateway API CRDs (v1.4.1, standard channel) into the homelab cluster using Flux GitOps workflow. This enables the existing Tailscale Gateway and HTTPRoute resources to function, allowing services like Longhorn UI to be exposed via the tailnet.

## Technical Context

**Language/Version**: YAML (Kubernetes manifests)
**Primary Dependencies**: Flux CD (GitRepository, Kustomization), kubernetes-sigs/gateway-api v1.4.1
**Storage**: N/A (CRDs are cluster-scoped resources)
**Testing**: kubectl verification commands, Flux status checks
**Target Platform**: Talos Linux Kubernetes cluster (v1.35.0)
**Project Type**: Infrastructure as Code (GitOps)
**Performance Goals**: CRD installation < 2 minutes, Gateway programmed < 5 minutes
**Constraints**: Standard channel CRDs only (no experimental), GitOps-compliant
**Scale/Scope**: Single cluster, 5 CRDs

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-check after Phase 1 design.*

| Principle | Status | Notes |
|-----------|--------|-------|
| I. GitOps-First | ✅ PASS | All changes via Git + Flux reconciliation |
| II. Declarative Configuration | ✅ PASS | YAML manifests with Kustomize |
| III. Secrets Management | ✅ N/A | No secrets required for CRD installation |
| IV. Immutable Infrastructure | ✅ N/A | CRDs don't affect node configuration |
| V. Documentation as Code | ✅ PASS | Spec, plan, quickstart documented |

**Gate Result**: PASS - Proceed to implementation

## Project Structure

### Documentation (this feature)

```text
specs/004-gateway-api/
├── plan.md              # This file
├── research.md          # Phase 0 output - installation method research
├── data-model.md        # Phase 1 output - Flux resources and CRD definitions
├── quickstart.md        # Phase 1 output - step-by-step installation guide
├── contracts/           # Phase 1 output
│   └── reconciliation.md # Flux reconciliation contract
├── checklists/
│   └── requirements.md  # Specification quality checklist
└── spec.md              # Feature specification
```

### Source Code (repository root)

```text
k8s/infrastructure/
├── gateway-api/              # NEW - Gateway API CRD installation
│   ├── kustomization.yaml    # Local Kustomize config
│   ├── gitrepository.yaml    # Flux GitRepository for upstream
│   └── kustomization-crds.yaml # Flux Kustomization for CRD installation
├── cilium/                   # Existing
├── longhorn/                 # Existing
├── tailscale/                # Existing (depends on Gateway API CRDs)
│   ├── gateway.yaml          # Uses Gateway CRD
│   └── httproute-longhorn.yaml # Uses HTTPRoute CRD
└── kustomization.yaml        # Updated to include gateway-api/ first
```

**Structure Decision**: Follow existing infrastructure directory pattern. Add `gateway-api/` as a new directory containing Flux resources for CRD management.

## Implementation Tasks

### Phase 1: Create Flux Resources

1. **Create gateway-api directory structure**
   - Create `k8s/infrastructure/gateway-api/`

2. **Create GitRepository resource**
   - File: `k8s/infrastructure/gateway-api/gitrepository.yaml`
   - References: `https://github.com/kubernetes-sigs/gateway-api`
   - Tag: `v1.4.1`

3. **Create Flux Kustomization for CRDs**
   - File: `k8s/infrastructure/gateway-api/kustomization-crds.yaml`
   - Path: `./config/crd/standard`
   - Prune: `false`

4. **Create local Kustomization**
   - File: `k8s/infrastructure/gateway-api/kustomization.yaml`
   - Resources: gitrepository.yaml, kustomization-crds.yaml

### Phase 2: Update Infrastructure Configuration

5. **Update infrastructure kustomization.yaml**
   - Add `gateway-api/` as first entry in resources list
   - Ensures CRDs are applied before tailscale resources

### Phase 3: Verification

6. **Commit and push changes**
   - Trigger Flux reconciliation

7. **Verify CRD installation**
   - Check GitRepository status
   - Check Kustomization status
   - Verify CRDs are registered

8. **Verify Tailscale integration**
   - Check Gateway status (Programmed=True)
   - Check HTTPRoute status (Accepted=True)
   - Test Longhorn UI access via tailnet

## Risk Assessment

| Risk | Likelihood | Impact | Mitigation |
|------|------------|--------|------------|
| Network connectivity to GitHub | Low | Medium | Flux retries with cached artifact |
| CRD version incompatibility | Low | High | Using stable v1.4.1, tested with Tailscale |
| Existing resources in error state | Medium | Low | CRD installation will resolve |
| Flux reconciliation delay | Low | Low | Manual reconciliation available |

## Rollback Plan

1. Remove `gateway-api/` from `k8s/infrastructure/kustomization.yaml`
2. Delete `k8s/infrastructure/gateway-api/` directory
3. Commit and push
4. Manually delete CRDs if needed (Flux won't prune them due to `prune: false`)

## Success Criteria Mapping

| Success Criteria | Verification Method |
|------------------|---------------------|
| SC-001: CRDs installed < 2 min | `kubectl get crd \| grep gateway` |
| SC-002: Flux Kustomization Ready | `kubectl get kustomization gateway-api-crds -n flux-system` |
| SC-003: Gateway programmed < 5 min | `kubectl get gateway -n tailscale -o jsonpath='{.status.conditions}'` |
| SC-004: Longhorn UI accessible | Browser test: `https://longhorn.<tailnet>.ts.net` |
| SC-005: Upgradeable via manifest | Update tag in gitrepository.yaml, verify reconciliation |
| SC-006: No API errors | `kubectl get gateway,httproute -A` returns without errors |
