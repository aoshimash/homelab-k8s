# Research: Manage Cilium with Flux

**Date**: 2026-01-02
**Branch**: `001-cilium-flux`
**Spec**: [spec.md](./spec.md)

## Decisions

### D1: Manage Cilium via Flux Helm Controller (HelmRepository + HelmRelease)

**Decision**: Use Flux `HelmRepository` + `HelmRelease` to manage the Cilium chart declaratively.

**Rationale**:
- Matches repository constitution: Helm charts deployed via Flux `HelmRelease`.
- Enables Git review/rollback and reconciliation-based drift correction.

**Alternatives considered**:
- Commit rendered Cilium manifests: rejected due to higher maintenance overhead and reduced alignment with the “Helm via Flux” standard for charts.

---

### D2: Git layout for Cilium resources

**Decision**: Add cluster infrastructure manifests under `k8s/infrastructure/cilium/` and have Flux apply them via a `Kustomization` resource stored under `k8s/flux/`.

**Rationale**:
- Keeps Flux system bootstrap (`k8s/flux/flux-system`) separate from infrastructure.
- Works with the existing Flux sync path (`./k8s/flux`) without needing to re-bootstrap.

**Alternatives considered**:
- Change Flux root `Kustomization.spec.path` from `./k8s/flux` to `./k8s`: rejected because `gotk-sync.yaml` is generated and changing it increases risk; can be revisited later if desired.

---

### D3: Migration strategy from manual Helm

**Decision**: Adopt the existing Helm release into Flux management (same release name/namespace) to avoid disruption.

**Rationale**:
- “Zero downtime/packet loss” target for migration.
- Avoids uninstall/reinstall churn for the CNI.

**Alternatives considered**:
- Uninstall and re-install via Flux: rejected as it risks node NotReady periods and workload disruption.

**Open operational dependency**:
- The current cluster’s installed Cilium version and values must be captured (e.g., from the existing Helm release) so the initial `HelmRelease` converges without changes.

---

### D4: Upgrade policy after migration

**Decision**: Pin chart version/config; no automatic upgrades. All changes via PR.

**Rationale**:
- Reduces surprise upgrades and keeps rollback straightforward.

---

### D5: Downtime policy

**Decision**: Target zero observed networking downtime/packet loss during migration.

**Rationale**:
- Networking is foundational; outages are high impact.
