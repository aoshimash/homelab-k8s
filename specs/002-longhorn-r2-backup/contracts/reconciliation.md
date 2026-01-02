# Contract: Flux-managed Longhorn reconciliation

## Purpose

Define what “correct” looks like when Longhorn is managed by Flux in this repository, so validation is repeatable and PRs can be reviewed against explicit expectations.

## Preconditions

- Flux controllers are installed and reconciling from `k8s/flux/`.
- Flux `Kustomization` responsible for `k8s/infrastructure/` is present and reconciling.
- SOPS + age key material is available to Flux as a Kubernetes Secret (referenced by Kustomization decryption configuration).

## Required resources (minimum set)

- A `HelmRepository` for the Longhorn chart source exists and is Ready.
- A `HelmRelease` for Longhorn exists and is Ready.
- Longhorn system components are running and Ready in their target namespace.
- An encrypted Secret manifest for R2 credentials exists in Git and decrypts successfully during reconciliation.

## Readiness & health signals

- `HelmRepository` condition Ready = True.
- `HelmRelease` condition Ready = True.
- Longhorn manager/UI components are Running/Ready.
- Backup credentials Secret exists in-cluster (post-decryption) and is readable by Longhorn.

## Invariants (must always hold after introduction)

- Git is the source of truth: drift is corrected by Flux reconciliation.
- Single-node baseline: default volume replica count is configured to 1.
- Cloudflare R2 is the configured backup target.
- Secrets are never committed in plaintext; only SOPS-encrypted `*.sops.yaml` secret manifests exist in Git.

## Failure handling contract

- If reconciliation fails due to invalid configuration or missing decryption key material, the failure is visible via Flux status and does not silently leave the cluster in an unknown partially-managed state.
- Rollback is performed by reverting the Git change that introduced the failure and letting Flux reconcile back to the previous state.
