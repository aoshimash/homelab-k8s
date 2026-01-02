# Quickstart: Longhorn + Cloudflare R2 backups (GitOps)

This quickstart describes the operator workflow to configure Longhorn with Cloudflare R2 backups in this repository, using SOPS + age encrypted secrets decrypted by Flux.

## Prerequisites

- You are on the feature branch: `002-longhorn-r2-backup`.
- Flux is installed and reconciling from `k8s/flux/`.
- You have Cloudflare R2:
  - Account ID (for the endpoint format)
  - Bucket name
  - Access key + secret key (scoped to the bucket)
- You have the age private key available locally to encrypt secrets (the private key is NOT committed to Git).

## Configure encrypted R2 credentials

1. Create a Kubernetes Secret manifest for the R2 credentials in the Longhorn namespace (name to be referenced by Longhorn configuration).
2. Encrypt it with SOPS (age), and commit only the encrypted `*.sops.yaml` file.

Notes:
- This repository already has `.sops.yaml` rules for Talos secrets. The Longhorn work will extend the rules to also cover Kubernetes secrets under `k8s/`.
- Flux must be configured to decrypt SOPS secrets during reconciliation (Kustomization decryption config + age key Secret in-cluster).

## Reconcile

1. Commit and push Git changes.
2. Wait for Flux reconciliation of the `infrastructure` kustomization to succeed.

## Validate provisioning

- Create a small test workload that requests a persistent volume, writes data, restarts, and reads the same data.
- Confirm the volume is healthy and not degraded by default in the single-node topology.

## Validate backup & restore

- Create or select a test volume with known test data.
- Confirm the daily backup schedule executes successfully.
- Trigger a restore into a new volume and verify the restored data matches.

## Troubleshooting

- If SOPS decryption fails, verify:
  - The age private key Secret exists in the cluster.
  - The Flux Kustomization includes `spec.decryption.provider: sops` and points to that Secret.
  - The encrypted secret files follow the `*.sops.yaml` naming convention and match `.sops.yaml` rules.
