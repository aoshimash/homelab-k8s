# Contract: Flux-managed Cilium reconciliation

## Purpose

Define what “correct” looks like when Cilium is managed by Flux, so validation is repeatable and PRs can be reviewed against explicit expectations.

## Preconditions

- Flux controllers are installed and reconciling from `k8s/flux/`.
- The cluster is already running Cilium (manual Helm) or can tolerate initial bootstrap sequencing.

## Required resources (minimum set)

- A `HelmRepository` for the Cilium chart source exists and is Ready.
- A `HelmRelease` for Cilium exists in `kube-system` and is Ready.

## Readiness & health signals

- `HelmRepository` condition Ready = True.
- `HelmRelease` condition Ready = True.
- Cilium pods in `kube-system` are Running/Ready (agent + operator).
- Node(s) are Ready.

## Invariants (must always hold after migration)

- No automatic upgrades: chart version and values remain pinned until changed by PR.
- Drift correction: if cluster state diverges from Git, reconciliation returns it to the declared state.
- Single manager: there is no competing imperative Helm workflow managing the same release.

## Failure handling contract

- If reconciliation fails due to invalid configuration, the failure is visible via Flux status and does not silently leave the cluster in an unknown partially-managed state.
- Rollback is performed by reverting the Git change that introduced the failure and reconciling.
