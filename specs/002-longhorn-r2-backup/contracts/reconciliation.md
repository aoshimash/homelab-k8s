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

## Validation checklist (operator-visible)

Run these checks after Flux reconciliation to confirm the system is in a known-good state:

- Confirm the age key Secret exists (out-of-band bootstrap):
  - `kubectl -n flux-system get secret sops-age`
- Confirm the Flux `infrastructure` Kustomization has decryption enabled:
  - `kubectl -n flux-system get kustomization infrastructure -o yaml`
  - Expected: `spec.decryption.provider: sops` and `spec.decryption.secretRef.name: sops-age`
- Confirm the Longhorn chart source is Ready:
  - `kubectl -n flux-system get helmrepository longhorn`
  - Expected: Ready=True
- Confirm the Longhorn release is Ready:
  - `kubectl -n longhorn-system get helmrelease longhorn`
  - Expected: Ready=True
- Confirm the decrypted credentials Secret exists in-cluster:
  - `kubectl -n longhorn-system get secret longhorn-r2-credentials`
  - Expected: Secret exists and includes the expected keys (post-decryption)

## Invariants (must always hold after introduction)

- Git is the source of truth: drift is corrected by Flux reconciliation.
- Single-node baseline: default volume replica count is configured to 1.
- Cloudflare R2 is the configured backup target.
- Secrets are never committed in plaintext; only SOPS-encrypted `*.sops.yaml` secret manifests exist in Git.

## Failure handling contract

- If reconciliation fails due to invalid configuration or missing decryption key material, the failure is visible via Flux status and does not silently leave the cluster in an unknown partially-managed state.
- Rollback is performed by reverting the Git change that introduced the failure and letting Flux reconcile back to the previous state.

## Decryption failure modes (must be documented and debuggable)

- **Missing age key Secret**:
  - Symptom: Flux Kustomization reports decryption errors; SOPS secrets do not apply.
  - Fix: Create `sops-age` Secret in `flux-system` from local `age.agekey`.
- **Wrong Kustomization `secretRef`**:
  - Symptom: Kustomization fails decryption even though the key Secret exists.
  - Fix: Ensure `spec.decryption.secretRef.name: sops-age` (namespace is `flux-system`).
- **Invalid SOPS file / MAC mismatch**:
  - Symptom: Apply fails with SOPS integrity errors.
  - Fix: Re-encrypt with correct recipients; do not edit ciphertext by hand.
- **Recipients mismatch (wrong age recipient)**:
  - Symptom: `sops` can encrypt, but Flux cannot decrypt with the provided key.
  - Fix: Ensure `.sops.yaml` recipients match the private key stored in-cluster.
- **Naming / rule mismatch (`*.sops.yaml`)**:
  - Symptom: Secret applies as ciphertext or is not picked up by decryption rules.
  - Fix: Ensure encrypted secrets are named `*.sops.yaml` and match `.sops.yaml` `creation_rules`.
