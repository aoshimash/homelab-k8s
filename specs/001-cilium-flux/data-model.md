# Data Model: Manage Cilium with Flux

This feature is infrastructure-as-code; the “data model” is the set of Kubernetes resources and their relationships.

## Entities (Kubernetes resources)

### Flux Source: HelmRepository (Cilium chart source)

- **Identity**: `(namespace, name)` (e.g., `flux-system/cilium`)
- **Attributes**:
  - Repository URL for the Cilium Helm chart
  - Polling interval

### Flux Helm: HelmRelease (Cilium installation)

- **Identity**: `(namespace, name)` (e.g., `kube-system/cilium`)
- **Key attributes**:
  - **releaseName**: Must match the existing manual Helm release for adoption during migration
  - **chart version**: Pinned (no automatic upgrades)
  - **values**: Must match the current running configuration to achieve a no-change handoff

### Flux Kustomize: Kustomization (apply infrastructure set)

- **Identity**: `(namespace, name)` (e.g., `flux-system/infrastructure`)
- **Attributes**:
  - `path` to `k8s/infrastructure/`
  - Prune enabled
  - Health checks / depends-on ordering (if required for bootstrap)

## Relationships

- `Kustomization` applies the YAML manifests in `k8s/infrastructure/`, including `HelmRepository` and `HelmRelease`.
- `HelmRelease` references the `HelmRepository` as its chart source.
- Cilium workloads run in `kube-system` and must become Ready before nodes/workloads are considered healthy.

## Invariants

- There is exactly one active source-of-truth for the Cilium Helm release after migration (Flux-managed).
- Chart version/config changes occur only via Git (PR).
- Migration targets zero observed downtime/packet loss.
