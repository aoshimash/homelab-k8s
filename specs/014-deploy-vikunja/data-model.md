# Data Model: Vikunja (GitOps Deployment)

**Feature**: 014-deploy-vikunja  
**Date**: 2026-01-10

This document describes the **declarative entities** managed in Git to deliver the Vikunja service, plus the core **application-level concepts** referenced in the feature spec.

## Application Concepts (from spec)

- **User**
  - Attributes: username, email (optional), credentials, membership in shared projects
  - Invariants:
    - No public self-registration (users are created by operator/admin)
    - No SSO for MVP (local accounts only)
- **Project**
  - Attributes: name, owner, shared members
  - Invariants:
    - Sharing grants edit access to shared users (collaborative editing)
- **Task**
  - Attributes: title, description, due date (optional), completion state, project reference
  - Invariants:
    - Tasks persist across restarts/upgrades

## Infrastructure Entities (Git-managed)

### Namespace
- **`Namespace/vikunja`**
  - Purpose: isolate application resources

### Ingress
- **`Ingress/vikunja`** (or chart-managed equivalent)
  - Constraints:
    - `ingressClassName: tailscale`
    - Annotation `tailscale.com/proxy-group: ingress-proxies`
  - Hostname:
    - `vikunja` (in-tailnet name)
  - Invariant:
    - Not directly reachable from the public internet

### Persistent Storage (Files/Attachments)
- **`PersistentVolumeClaim/vikunja-files`**
  - StorageClass: `longhorn`
  - Annotation:
    - `recurring-job-selector.longhorn.io: '[{"name":"backup-daily","isGroup":false}]'`
  - Invariants:
    - Must be mounted by the Vikunja workload for file storage (attachments)
    - Must be covered by daily backups (Longhorn recurring backup)

### Database (CloudNativePG)
- **`Secret/postgres-vikunja-user`** (SOPS-encrypted, in `postgres` namespace)
  - Type: `kubernetes.io/basic-auth` (recommended by CNPG docs)
  - Keys: `username`, `password`
  - Invariant:
    - Secret must be encrypted at rest in Git (`*.sops.yaml`)

- **`Cluster/postgres-cluster`** (existing)
  - Extended with:
    - `spec.managed.roles[]` entry for the Vikunja DB role, referencing the passwordSecret above

- **`Database/postgres-cluster-vikunja`** (new)
  - Name: `vikunja`
  - Owner: vikunja DB role
  - Invariant:
    - Database exists declaratively and is recreated predictably if needed

### Application Secrets
- **`Secret/vikunja`** (SOPS-encrypted, in `vikunja` namespace)
  - Contains:
    - JWT secret (and any other sensitive application config)
    - DB password (either duplicated safely for app consumption, or provided via dedicated app secret)
  - Invariants:
    - Must not enable registration
    - Must not configure public exposure

### Helm Release (Application Packaging)
- **`HelmRepository/vikunja`**
  - Points to the official Vikunja chart repository
- **`HelmRelease/vikunja`**
  - Configures:
    - External PostgreSQL (CNPG service DNS)
    - Persistence for files (use existing PVC)
    - Registration disabled
    - Ingress aligned with Tailscale pattern (either chart values or separate Ingress)

## State & Lifecycle Notes

- **Reconciliation**: Flux continuously reconciles all Git-defined resources.
- **Upgrades**: Upgrading Vikunja is done by chart version bump in Git; must preserve DB and PVC.
- **Disaster recovery**:
  - DB restored from CNPG backups (R2)
  - Files restored from Longhorn backups

