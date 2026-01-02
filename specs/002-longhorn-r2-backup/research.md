# Research: Introduce Longhorn with R2 Backups

This document records planning decisions and rationale for installing Longhorn on a single-node cluster and configuring Cloudflare R2 as the backup target, with SOPS + age secrets decrypted by Flux during reconciliation.

## Decisions

### Decision 1 — Deploy Longhorn via Flux Helm controller

- **Decision**: Manage Longhorn with Flux `HelmRepository` + `HelmRelease` under `k8s/infrastructure/longhorn/`.
- **Rationale**: Aligns with repository constitution (GitOps-first, declarative configuration) and keeps upgrades/reverts PR-based.
- **Alternatives considered**:
  - Install via imperative Helm CLI: rejected by GitOps rules.
  - Static manifests only: increases drift risk and maintenance burden.

### Decision 2 — Single-node tuning: default replica count = 1

- **Decision**: Configure Longhorn default replica count to 1 for new volumes.
- **Rationale**: Prevents “degraded-by-design” states when only one node exists; matches clarified requirement.
- **Alternatives considered**:
  - Default replicas > 1: rejected because replicas cannot be scheduled on a single node.
  - Workload-specific per volume only: rejected because it increases operational burden for a baseline homelab setup.

### Decision 3 — Cloudflare R2 as S3-compatible backup target

- **Decision**: Use Cloudflare R2 (S3-compatible) as the Longhorn backup target.
- **Rationale**: Off-cluster object storage enables recovery from node loss; R2 offers S3 API compatibility and cost advantages for homelabs.
- **Notes**:
  - The R2 endpoint is account-specific (format includes an account ID).
  - During implementation, validate the exact Longhorn backup target URI and any required endpoint configuration against current Longhorn documentation/UI.
- **Alternatives considered**:
  - Local/NAS-only backups: rejected due to shared failure domain with the cluster.
  - Another S3 provider: not selected for this feature scope.

### Decision 4 — Daily automatic backups with 30-day retention

- **Decision**: Configure daily scheduled backups and retain backups for 30 days.
- **Rationale**: Balances operational simplicity, recovery window, and object-storage cost.
- **Alternatives considered**:
  - Hourly backups: rejected as unnecessary for initial homelab scope and cost.
  - Weekly backups: rejected as too large an RPO for common mistakes.

### Decision 5 — Unchanged-day behavior: run with minimal overhead

- **Decision**: Scheduled backups may still execute on schedule even if data is unchanged, but should minimize additional storage consumption and completion time.
- **Rationale**: Keeps “backup is working” signal while avoiding full re-copying of unchanged data; avoids forcing implementation-specific skip semantics.
- **Alternatives considered**:
  - Hard skip when unchanged: rejected as it can be implementation-dependent and harder to validate consistently.

### Decision 6 — Secret handling: SOPS + age encrypted in Git, decrypted by Flux

- **Decision**: Store R2 credentials in SOPS-encrypted Kubernetes Secret manifests (`*.sops.yaml`) committed to Git, and enable Flux `Kustomization.spec.decryption` with provider `sops` and an age private key Secret reference.
- **Rationale**: Satisfies constitution “Secrets Management” while keeping GitOps reproducibility. Decryption happens at reconciliation time; plaintext never lands in Git.
- **Repository alignment**:
  - `.sops.yaml` already defines age recipients for Talos secrets; extend it with a rule for `k8s/**/*.sops.yaml`.
- **Alternatives considered**:
  - External secret manager: out of scope for this repo and feature.

## Risks & Mitigations

- **Single-node availability**: With replicas=1, node loss implies volume unavailability.
  - **Mitigation**: Use R2 backups + restore workflow; document expectations.
- **Credential exposure risk**:
  - **Mitigation**: Enforce `*.sops.yaml` convention; enable Flux decryption; validate no plaintext secrets are present.
- **R2 endpoint/compat differences**:
  - **Mitigation**: Validate exact Longhorn S3 settings during implementation and record tested configuration in repository docs.
