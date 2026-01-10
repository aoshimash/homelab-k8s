# Quickstart: Deploy Vikunja (GitOps)

**Feature**: 014-deploy-vikunja  
**Date**: 2026-01-10

## Prerequisites

- Flux CD is reconciling this repository’s `k8s/` directory.
- CloudNativePG operator + the existing PostgreSQL cluster in `k8s/configs/postgres/` are healthy.
- Longhorn is installed and the recurring backup job `backup-daily` exists.
- Tailscale Ingress is available (existing `ingressClassName: tailscale` pattern).

## 1. Add database role + database (declarative)

1. Create a SOPS-encrypted Secret in `k8s/configs/postgres/secret-vikunja-db.sops.yaml` containing the DB role credentials.
2. Update `k8s/configs/postgres/cluster.yaml` to add a `spec.managed.roles[]` entry for the Vikunja role referencing that Secret.
3. Add a CNPG `Database` CRD (e.g. `k8s/configs/postgres/database-vikunja.yaml`) to create the `vikunja` database owned by that role.
4. Commit and let Flux reconcile.

## 2. Add Vikunja application resources

1. Create `k8s/apps/vikunja/namespace.yaml` and wire it into `k8s/apps/kustomization.yaml`.
2. Create a Longhorn PVC for file storage (attachments) with the daily backup annotation.
3. Create SOPS-encrypted Secrets for:
   - Vikunja JWT secret
   - DB password (and any other sensitive config needed by the chart)
4. Create `HelmRepository` + `HelmRelease` for the official Vikunja chart, configured for:
   - External PostgreSQL (CNPG service DNS)
   - Existing PVC for files
   - Registration disabled
   - Ingress aligned with the Tailscale pattern (either chart values or a separate `Ingress` manifest)
5. Commit and let Flux reconcile.

## 3. Smoke test

From a device inside the trusted network/VPN:
- Open `https://vikunja/` (or the configured host in the Ingress)
- Verify the login page loads

## 4. Create the first user (admin-created users only)

After the pod is Ready, create users via the Vikunja CLI inside the container (operator action).

Example (commands may vary depending on chart naming):

```bash
kubectl -n vikunja get pods
kubectl -n vikunja exec -it deploy/vikunja -- ./vikunja user create --email you@example.com --user you --password 'CHANGEME'
```

Then sign in and create a test project/task.

## 5. Backup/restore sanity

- Confirm CNPG scheduled backups run successfully.
- Confirm Longhorn daily backups exist for the Vikunja files volume.
- Perform a restore drill periodically per `contracts/backup-restore.md`.

