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

## Bootstrap the Flux SOPS age key Secret (one-time, out-of-band)

Flux needs the **age private key** available in-cluster to decrypt `*.sops.yaml` files during reconciliation.

Create the Secret in `flux-system` **from your local** `age.agekey` (do not commit the private key to Git):

```bash
kubectl -n flux-system create secret generic sops-age --from-file=age.agekey=age.agekey
```

## Configure encrypted R2 credentials

1. Create a Kubernetes Secret manifest for the R2 credentials in the Longhorn namespace (name to be referenced by Longhorn configuration).
2. Encrypt it with SOPS (age), and commit only the encrypted `*.sops.yaml` file.

Notes:
- This repository already has `.sops.yaml` rules for Talos secrets and Kubernetes secrets under `k8s/` (SOPS + age).
- Flux must be configured to decrypt SOPS secrets during reconciliation (Kustomization decryption config + `sops-age` Secret in-cluster).

For this cluster:
- Bucket: `homelab-longhorn-backups`
- Endpoint: `https://<R2_ACCOUNT_ID>.r2.cloudflarestorage.com`

## Validate manifests render (local)

Before pushing, validate that the manifests render cleanly:

```bash
kustomize build k8s/flux >/dev/null
kustomize build k8s/infrastructure >/dev/null
echo "ok"
```

Expected outcome:
- Both commands exit with code 0 (no missing files, syntax errors, or invalid kustomizations).

## Reconcile

1. Commit and push Git changes.
2. Wait for Flux reconciliation of the `infrastructure` kustomization to succeed.

## Validate provisioning

- Option A (GitOps smoke): temporarily enable `k8s/infrastructure/longhorn/smoke/` by uncommenting it in `k8s/infrastructure/longhorn/kustomization.yaml`.
- Option B (manual apply for validation only): apply the smoke kustomization:
  - `kubectl apply -k k8s/infrastructure/longhorn/smoke`
- Confirm:
  - The PVC is Bound and the pod mounts it.
  - Restarting the pod does not lose the written value.
  - Longhorn volume health is not “degraded-by-design” on a single-node cluster (replicas default to 1).

## Validate backup & restore

- Create or select a test volume with known test data.
- Confirm the daily backup schedule executes successfully and a backup artifact becomes available.
- Restore into a new volume and verify:
  - The restored volume is attachable/mountable.
  - The restored content matches the original.

## Verify no plaintext secrets are committed

Run a quick guardrail check before opening a PR:

```bash
git grep -nE 'AWS_SECRET_ACCESS_KEY|secretAccessKey|BEGIN (RSA|OPENSSH) PRIVATE KEY' -- k8s || true
git grep -nE 'kind:\\s*Secret' -- 'k8s/**/*.yaml' ':!k8s/**/*.sops.yaml' || true
```

Expected outcome:
- No plaintext credential values are present in Git (only SOPS-encrypted `*.sops.yaml` secrets).

## Troubleshooting

- If SOPS decryption fails, verify:
  - The age private key Secret exists in the cluster.
  - The Flux Kustomization includes `spec.decryption.provider: sops` and points to that Secret.
  - The encrypted secret files follow the `*.sops.yaml` naming convention and match `.sops.yaml` rules.
