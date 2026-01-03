# Implementation Plan: kube-state-metrics

**Branch**: `006-kube-metrics` | **Date**: 2026-01-03 | **Spec**: [spec.md](./spec.md)  
**Input**: Feature specification from `/specs/006-kube-metrics/spec.md`

## Summary

Deploy kube-state-metrics to the homelab Kubernetes cluster to collect Kubernetes resource state metrics (Deployments, Pods, Nodes, etc.) and integrate with the existing Grafana Alloy pipeline to send metrics to Grafana Cloud. The implementation follows GitOps patterns consistent with other infrastructure components.

## Technical Context

**Language/Version**: YAML (Kubernetes manifests)  
**Primary Dependencies**: kube-state-metrics Helm chart (prometheus-community/kube-state-metrics v5.37.0), Grafana Alloy (existing)  
**Storage**: N/A (stateless metrics generator)  
**Testing**: Manual verification via Grafana Cloud queries, kubectl checks  
**Target Platform**: Kubernetes (Talos Linux homelab cluster)  
**Project Type**: Infrastructure/GitOps  
**Performance Goals**: Metrics available in Grafana Cloud within 5 minutes of resource state changes  
**Constraints**: Single-replica deployment, minimal resource footprint for small homelab  
**Scale/Scope**: Small homelab cluster (3-5 nodes)

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-check after Phase 1 design.*

| Principle | Status | Evidence |
|-----------|--------|----------|
| I. GitOps-First | ✅ PASS | All manifests committed to Git, deployed via Flux HelmRelease |
| II. Declarative Configuration | ✅ PASS | Using Kustomize + HelmRelease pattern |
| III. Secrets Management | ✅ PASS | No new secrets required (kube-state-metrics uses ServiceAccount) |
| IV. Immutable Infrastructure | ✅ PASS | No node-level changes required |
| V. Documentation as Code | ✅ PASS | Operational docs will be added to `docs/` |

**Gate Result**: ✅ PASS - All constitutional principles satisfied.

## Project Structure

### Documentation (this feature)

```text
specs/006-kube-metrics/
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
k8s/infrastructure/
├── kustomization.yaml              # UPDATE: Add kube-state-metrics reference
└── kube-state-metrics/             # NEW: kube-state-metrics manifests
    ├── kustomization.yaml          # Kustomize resource list
    ├── helmrepository.yaml         # prometheus-community Helm repo
    └── helmrelease.yaml            # kube-state-metrics HelmRelease

k8s/infrastructure/grafana-alloy/
└── helmrelease.yaml                # UPDATE: Add kube-state-metrics scrape config

docs/
└── kube-state-metrics.md           # NEW: Operational documentation
```

**Structure Decision**: Follow existing infrastructure component pattern (cilium/, longhorn/, tailscale/, grafana-alloy/). kube-state-metrics will be deployed to the existing `monitoring` namespace (shared with Grafana Alloy) to avoid namespace proliferation.

## Complexity Tracking

> No violations - implementation follows established patterns with minimal additions.

| Aspect | Decision | Rationale |
|--------|----------|-----------|
| Namespace | Use existing `monitoring` | Shared with Grafana Alloy, reduces namespace proliferation |
| HelmRepository | New `prometheus-community` | Official source for kube-state-metrics chart |
| Secrets | None required | Uses Kubernetes ServiceAccount for API access |

## Implementation Phases

### Phase 1: Deploy kube-state-metrics (User Story 2)

1. Create `k8s/infrastructure/kube-state-metrics/` directory
2. Add HelmRepository for prometheus-community
3. Add HelmRelease with default configuration
4. Update `k8s/infrastructure/kustomization.yaml`
5. Verify deployment via Flux reconciliation

### Phase 2: Integrate with Grafana Alloy (User Story 3)

1. Update Grafana Alloy config to scrape kube-state-metrics
2. Use Kubernetes service discovery for endpoint
3. Apply `cluster=homelab` label via relabeling
4. Verify metrics in Grafana Cloud

### Phase 3: Verification & Documentation (User Story 1)

1. Verify kube_* metrics in Grafana Cloud
2. Test metric accuracy (compare with kubectl output)
3. Add operational documentation
