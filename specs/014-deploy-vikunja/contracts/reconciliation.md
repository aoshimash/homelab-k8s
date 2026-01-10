# Contract: Reconciliation (Vikunja)

**Feature**: 014-deploy-vikunja  
**Date**: 2026-01-10

## Goal

After a Git commit is merged/applied, Flux reconciliation MUST converge to a healthy Vikunja service that is reachable via Tailscale Ingress and persists data across restarts.

## Managed Resources (expected to exist)

- `k8s/apps/vikunja/**` (namespace + app manifests)
- `k8s/configs/postgres/**` additions:
  - CNPG managed role for Vikunja
  - CNPG Database CRD for Vikunja

## Health Signals (must be true)

- **Flux**
  - Relevant `Kustomization` resources are Ready
  - `HelmRelease/vikunja` is Ready (if Helm-based deployment is used)
- **Kubernetes**
  - Vikunja pods are Running and Ready
  - Service endpoints exist (no crashloop)
  - PVC is Bound
- **Ingress**
  - `Ingress/vikunja` exists (either chart-managed or repo-managed)
  - Host `vikunja` is reachable from the trusted network/VPN
- **Application**
  - Login page loads
  - Admin-created user can sign in and create a task

## Reconciliation Steps (operator checklist)

1. Confirm Flux controllers are healthy.
2. Confirm `Kustomization` / `HelmRelease` status for Vikunja is Ready.
3. Confirm CNPG database and role objects are Ready/Applied.
4. Confirm PVC is bound and mounted by the workload.
5. Confirm Ingress is created and resolves within Tailscale.
6. Run smoke test (sign in, create project/task).

## Failure Modes (expected handling)

- **DB not reachable / misconfigured credentials**:
  - Pods may be Running but app returns errors; logs must show actionable DB errors.
  - Fix by correcting Secrets/DB resources in Git and letting Flux reconcile.
- **PVC not bound**:
  - Workload may not start or attachments fail.
  - Fix storageClass/size or Longhorn status; keep changes declarative.
- **Ingress not reachable**:
  - Verify `ingressClassName: tailscale` and proxy-group annotation.
  - Ensure access only via trusted network/VPN.

