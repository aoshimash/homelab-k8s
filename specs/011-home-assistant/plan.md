# Implementation Plan: Home Assistant導入とSwitchBot Hub 2のMatter連携

**Branch**: `011-home-assistant` | **Date**: 2026-01-04 | **Spec**: [spec.md](./spec.md)
**Input**: Feature specification from `/specs/011-home-assistant/spec.md`

## Summary

Home AssistantをKubernetesクラスターにデプロイし、SwitchBot Hub 2とMatterプロトコルでローカル通信する。hostNetworkモードでmDNS/IPv6リンクローカル通信を有効化し、Tailscale Ingress経由で外部アクセスを提供する。オートメーション設定はConfigMapとしてGit管理する。

## Technical Context

**Language/Version**: YAML (Kubernetes manifests), Home Assistant 2024.x+
**Primary Dependencies**: Home Assistant Container, Matter integration (built-in)
**Storage**: Longhorn PVC (5Gi for /config)
**Testing**: Manual verification via Home Assistant UI, kubectl commands
**Target Platform**: Kubernetes on Talos Linux (single node cluster)
**Project Type**: Kubernetes application deployment (GitOps with Flux)
**Performance Goals**: WebUI access within 5 seconds, Matter device response within 1 second
**Constraints**: hostNetwork required for Matter/mDNS, single node (homelab-node-01)
**Scale/Scope**: Single Home Assistant instance, 1 SwitchBot Hub 2 device

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-check after Phase 1 design.*

| Principle | Status | Evidence |
|-----------|--------|----------|
| I. GitOps-First | ✅ PASS | All manifests in `k8s/apps/home-assistant/`, Flux reconciliation |
| II. Declarative Configuration | ✅ PASS | Kustomize structure, no imperative commands |
| III. Secrets Management | ✅ PASS | No secrets required for basic deployment |
| IV. Immutable Infrastructure | ✅ PASS | No node-level changes required |
| V. Documentation as Code | ✅ PASS | docs/home-assistant.md will be created |

**Gate Result**: ✅ PASS - All principles satisfied

## Project Structure

### Documentation (this feature)

```text
specs/011-home-assistant/
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
k8s/apps/home-assistant/
├── app/
│   ├── deployment.yaml      # Home Assistant Deployment with hostNetwork
│   ├── service.yaml         # ClusterIP Service (port 8123)
│   ├── ingress.yaml         # Tailscale Ingress (hostname: home-assistant)
│   ├── pvc.yaml             # Longhorn PVC (5Gi)
│   ├── configmap.yaml       # Sample automation configuration
│   └── kustomization.yaml   # App-level kustomization
├── namespace.yaml           # home-assistant namespace
└── kustomization.yaml       # Root kustomization

docs/
└── home-assistant.md        # Operational documentation
```

**Structure Decision**: Single application structure following existing `audiobookshelf` pattern in `k8s/apps/`. Uses Kustomize for manifest management with Flux GitOps reconciliation.

## Complexity Tracking

No violations requiring justification. Design follows existing patterns in the codebase.
