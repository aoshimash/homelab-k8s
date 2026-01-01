<!--
  Sync Impact Report
  ==================
  Version change: N/A → 1.0.0 (Initial creation)

  Added Principles:
  - I. GitOps-First
  - II. Declarative Configuration
  - III. Secrets Management
  - IV. Immutable Infrastructure
  - V. Documentation as Code

  Added Sections:
  - Technology Stack
  - Development Workflow
  - Governance

  Templates requiring updates:
  - .specify/templates/plan-template.md: ✅ No changes required (generic template)
  - .specify/templates/spec-template.md: ✅ No changes required (generic template)
  - .specify/templates/tasks-template.md: ✅ No changes required (generic template)

  Follow-up TODOs: None
-->

# homelab-k8s Constitution

## Core Principles

### I. GitOps-First

All cluster state and configuration MUST be defined in Git. Changes to the cluster MUST be applied through Git commits and automated reconciliation via Flux CD.

**Rules:**
- Infrastructure changes MUST be committed to Git before being applied to the cluster
- Manual `kubectl apply` or `helm install` commands are prohibited for persistent changes
- Flux CD MUST reconcile the cluster state from the `k8s/` directory
- Emergency changes MUST be backported to Git immediately and reconciled

**Rationale:** GitOps provides audit trails, rollback capability, and consistent reproducibility. The Git repository is the single source of truth.

### II. Declarative Configuration

All infrastructure and application configurations MUST be expressed declaratively using YAML manifests or configuration files.

**Rules:**
- Talos Linux configuration MUST be managed via `talconfig.yaml` and Talhelper
- Kubernetes manifests MUST use Kustomize for environment-specific overlays
- Helm charts MUST be deployed via Flux HelmRelease resources, not imperative `helm install`
- Configuration drift MUST be automatically corrected by Flux reconciliation

**Rationale:** Declarative configuration enables idempotent operations, self-healing, and clear visibility into desired state.

### III. Secrets Management

All secrets MUST be encrypted at rest using SOPS with Age encryption before being committed to Git.

**Rules:**
- Secrets MUST be encrypted using SOPS before committing to the repository
- The Age private key (`age.agekey`) MUST NOT be committed to Git
- Encrypted secrets MUST follow the naming convention `*.sops.yaml`
- Flux Kustomizations handling secrets MUST include SOPS decryption configuration
- `.sops.yaml` MUST define explicit path patterns for encryption rules

**Rationale:** Secrets in Git enable GitOps workflows while encryption prevents exposure of sensitive data.

### IV. Immutable Infrastructure

Cluster nodes MUST be treated as immutable. Configuration changes MUST be applied through Talos machine configuration, not runtime modifications.

**Rules:**
- Node configuration changes MUST be applied via `talosctl apply-config`
- SSH access to nodes is NOT available (Talos design principle)
- System extensions MUST be included in the Talos image schematic, not installed at runtime
- Cluster upgrades MUST be performed via Talos upgrade mechanisms, not in-place package updates

**Rationale:** Immutability ensures consistency, security, and reproducibility across cluster operations.

### V. Documentation as Code

All operational procedures and architectural decisions MUST be documented in the `docs/` directory and kept in sync with actual configurations.

**Rules:**
- Bootstrap procedures MUST be documented with exact commands
- Configuration changes MUST be accompanied by documentation updates
- Tool versions and dependencies MUST be explicitly documented
- Troubleshooting guides MUST be maintained for common issues

**Rationale:** Documentation enables knowledge transfer, disaster recovery, and reduces operational risk.

## Technology Stack

This project uses the following technologies for declarative infrastructure management:

| Component | Technology | Purpose |
|-----------|------------|---------|
| Operating System | Talos Linux | Immutable, secure Kubernetes OS |
| Configuration Management | Talhelper + SOPS | Declarative Talos configuration with encrypted secrets |
| GitOps | Flux CD | Continuous reconciliation of cluster state |
| CNI | Cilium | eBPF-based networking with kube-proxy replacement |
| Package Management | Kustomize + Helm | Kubernetes manifest management |
| Secrets Encryption | SOPS + Age | Git-compatible secrets encryption |

**Directory Structure:**

| Path | Purpose |
|------|---------|
| `infra/talos/` | Talos Linux configuration (applied outside cluster) |
| `k8s/flux/` | Flux system components |
| `k8s/infrastructure/` | Cluster infrastructure (CNI, ingress, cert-manager) |
| `k8s/apps/` | User applications |
| `docs/` | Operational documentation |

## Development Workflow

### Adding New Applications

1. Create application manifests in `k8s/apps/<app-name>/`
2. Add Kustomize resources referencing the application
3. If secrets are required, create SOPS-encrypted secret files
4. Commit changes to Git
5. Flux automatically reconciles the new application

### Modifying Cluster Configuration

1. Update `infra/talos/talconfig.yaml` as needed
2. Regenerate Talos configs: `talhelper genconfig`
3. Apply configuration: `talosctl apply-config`
4. Commit configuration changes to Git

### Secret Rotation

1. Update the secret value in the SOPS-encrypted file
2. Re-encrypt if necessary: `sops -e -i <file>`
3. Commit the encrypted file
4. Flux automatically reconciles the updated secret

## Governance

This constitution defines the operational principles for the homelab-k8s repository. All changes to cluster infrastructure and applications MUST comply with these principles.

**Amendment Process:**
1. Propose changes via pull request with rationale
2. Document impact on existing workflows
3. Update affected documentation
4. Merge after review

**Compliance:**
- All pull requests MUST be reviewed against these principles
- Automated reconciliation failures MUST be investigated and resolved
- Deviations from principles MUST be documented with justification

**Version**: 1.0.0 | **Ratified**: 2026-01-02 | **Last Amended**: 2026-01-02
