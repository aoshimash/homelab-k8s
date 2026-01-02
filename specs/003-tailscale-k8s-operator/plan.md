# Implementation Plan: Tailscale Kubernetes Operator Installation

**Branch**: `003-tailscale-k8s-operator` | **Date**: 2026-01-02 | **Spec**: [spec.md](./spec.md)
**Input**: Feature specification from `/specs/003-tailscale-k8s-operator/spec.md`

## Summary

Install the Tailscale Kubernetes Operator via Helm using Flux GitOps workflow. The operator will be deployed with SOPS-encrypted OAuth credentials. Longhorn Web UI will be exposed via Tailscale Gateway API (Gateway + HTTPRoute) as a validation target.

## Technical Context

**Infrastructure Type**: Kubernetes GitOps (Flux CD)
**Primary Dependencies**: Flux CD, Helm, SOPS, Tailscale Operator Helm Chart
**Storage**: N/A (operator is stateless)
**Testing**: Manual verification via kubectl and Tailscale admin console
**Target Platform**: Talos Linux Kubernetes cluster
**Project Type**: Infrastructure configuration (Kubernetes manifests)
**Performance Goals**: Operator Pod running within 5 minutes of reconciliation
**Constraints**: None (Gateway API does not require Cilium socket LB bypass)
**Scale/Scope**: Single-node homelab cluster

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-check after Phase 1 design.*

| Principle | Status | Notes |
|-----------|--------|-------|
| I. GitOps-First | ✅ PASS | All resources managed via Flux HelmRelease and Kustomization |
| II. Declarative Configuration | ✅ PASS | Helm chart deployed via Flux HelmRelease, not imperative commands |
| III. Secrets Management | ✅ PASS | OAuth credentials encrypted with SOPS before commit |
| IV. Immutable Infrastructure | ✅ PASS | No node-level changes required |
| V. Documentation as Code | ✅ PASS | Spec and plan documented in specs/ directory |

**Gate Result**: ✅ All principles satisfied. Proceeding to Phase 0.

## Project Structure

### Documentation (this feature)

```text
specs/003-tailscale-k8s-operator/
├── plan.md              # This file
├── research.md          # Phase 0 output
├── data-model.md        # Phase 1 output
├── quickstart.md        # Phase 1 output
├── contracts/           # Phase 1 output
│   └── reconciliation.md
└── tasks.md             # Phase 2 output (created by /speckit.tasks)
```

### Source Code (repository root)

```text
k8s/infrastructure/tailscale/
├── kustomization.yaml           # Kustomize entrypoint
├── namespace.yaml               # tailscale namespace
├── helmrepository.yaml          # Tailscale Helm chart source
├── helmrelease.yaml             # Tailscale Operator deployment
├── secret-oauth.sops.yaml       # SOPS-encrypted OAuth credentials
├── gateway.yaml                 # Gateway API - Tailscale gateway
└── httproute-longhorn.yaml      # Gateway API - HTTPRoute for Longhorn UI

k8s/infrastructure/kustomization.yaml  # Updated to include tailscale/
```

**Structure Decision**: Following existing patterns from `cilium/` and `longhorn/` directories. All Tailscale resources are grouped under `k8s/infrastructure/tailscale/` with Flux managing reconciliation.

## Complexity Tracking

No violations to justify. Implementation follows established patterns.
