# Feature Specification: Tailscale Kubernetes Operator Installation

**Feature Branch**: `003-tailscale-k8s-operator`
**Created**: 2026-01-02
**Status**: Draft
**Input**: User description: "Install Tailscale Kubernetes Operator via Helm"

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Install Tailscale Operator via Helm (Priority: P1)

As a cluster administrator, I want to install the Tailscale Kubernetes Operator using Helm so that I can manage Tailscale resources within my Kubernetes cluster and expose services to my tailnet via Ingress.

**Why this priority**: This is the core functionality - without the operator installed, no Tailscale integration features can be used. It enables Ingress functionality to expose cluster services to the tailnet.

**Independent Test**: Can be fully tested by verifying the operator Pod is running and the operator has joined the tailnet. Delivers the foundation for Tailscale Ingress functionality.

**Acceptance Scenarios**:

1. **Given** a Kubernetes cluster with Helm installed and valid Tailscale OAuth credentials, **When** the Tailscale Operator Helm chart is installed, **Then** the operator Pod should be running in the `tailscale` namespace
2. **Given** the Tailscale Operator is installed, **When** checking the Tailscale admin console, **Then** a device named `tailscale-operator` should appear with the `tag:k8s-operator` tag
3. **Given** the Tailscale Operator is installed, **When** running `kubectl get pods -n tailscale`, **Then** the operator Pod should show `Running` status with `1/1` ready containers
4. **Given** the Tailscale Operator is installed with Ingress configured for Longhorn UI, **When** accessing the Longhorn UI via tailnet hostname, **Then** the Longhorn dashboard should be accessible from any device on the tailnet

---

### User Story 2 - Manage Operator via GitOps (Priority: P2)

As a cluster administrator, I want the Tailscale Operator installation to be managed via Flux (GitOps) so that the configuration is version-controlled and automatically reconciled.

**Why this priority**: GitOps management ensures consistency, auditability, and automated deployment, which aligns with the existing infrastructure management approach using Flux.

**Independent Test**: Can be tested by pushing changes to the repository and verifying Flux reconciles the Tailscale Operator resources automatically.

**Acceptance Scenarios**:

1. **Given** Flux is running in the cluster, **When** the Tailscale HelmRelease manifest is committed to the repository, **Then** Flux should automatically install the Tailscale Operator
2. **Given** the Tailscale Operator is managed by Flux, **When** checking `kubectl get helmrelease -n tailscale`, **Then** the HelmRelease should show `Ready` status
3. **Given** the Tailscale Operator is managed by Flux, **When** the HelmRelease values are updated in Git, **Then** Flux should reconcile the changes automatically

---

### User Story 3 - Secure Credential Management (Priority: P2)

As a cluster administrator, I want the Tailscale OAuth credentials to be securely stored using SOPS encryption so that sensitive credentials are not exposed in the Git repository.

**Why this priority**: Security is critical for credential management. OAuth credentials must be encrypted at rest in the repository to prevent unauthorized access to the tailnet.

**Independent Test**: Can be tested by verifying the encrypted secret file cannot be read without the appropriate decryption key, and Flux can decrypt and apply it.

**Acceptance Scenarios**:

1. **Given** a SOPS-encrypted secret file containing Tailscale OAuth credentials, **When** the file is committed to the repository, **Then** the credentials should not be readable in plain text
2. **Given** Flux with SOPS decryption configured, **When** the encrypted secret is applied, **Then** Flux should decrypt and create the Kubernetes Secret successfully
3. **Given** the Tailscale Operator is running, **When** checking the operator logs, **Then** there should be no authentication errors related to OAuth credentials

---

### Edge Cases

- What happens when the OAuth credentials are invalid or expired?
- How does the system handle network connectivity issues to Tailscale's coordination server?
- What happens if the `tailscale` namespace already exists with conflicting resources?
- How does the operator behave during cluster upgrades or node restarts?

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: System MUST install the Tailscale Kubernetes Operator using the official Helm chart from `https://pkgs.tailscale.com/helmcharts`
- **FR-002**: System MUST deploy the operator in a dedicated `tailscale` namespace
- **FR-003**: System MUST configure the operator with valid OAuth client credentials (client ID and client secret)
- **FR-004**: System MUST encrypt OAuth credentials using SOPS before storing in the Git repository
- **FR-005**: System MUST integrate with the existing Flux GitOps workflow for deployment and reconciliation
- **FR-006**: System MUST tag the operator device with `tag:k8s-operator` in the Tailscale admin console
- **FR-007**: System MUST allow configuration of the operator hostname (default: `tailscale-operator`)
- **FR-008**: System MUST configure Cilium socket load balancer bypass for Tailscale proxy Pods' namespaces (required because Cilium runs in kube-proxy replacement mode)
- **FR-009**: System MUST pin the Helm chart to a specific version for reproducibility and planned upgrades
- **FR-010**: System MUST configure Tailscale Ingress for Longhorn Web UI as a validation target

### Key Entities

- **HelmRepository**: Represents the Tailscale Helm chart repository source
- **HelmRelease**: Defines the Tailscale Operator deployment configuration including version and values
- **Secret**: Contains the encrypted OAuth client credentials (client ID and client secret)
- **Namespace**: The `tailscale` namespace where all Tailscale resources are deployed
- **Kustomization**: Flux Kustomization resource that manages the Tailscale infrastructure components

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: Tailscale Operator Pod reaches `Running` state within 5 minutes of Flux reconciliation
- **SC-002**: Operator device appears in Tailscale admin console with correct hostname and tags within 2 minutes of Pod startup
- **SC-003**: Flux HelmRelease shows `Ready=True` condition after successful deployment
- **SC-004**: OAuth credentials remain encrypted in the Git repository (verified by inspecting raw file contents)
- **SC-005**: Operator can be upgraded by updating the chart version in the HelmRelease manifest without manual intervention
- **SC-006**: Longhorn Web UI is accessible via tailnet hostname from any device connected to the tailnet

## Clarifications

### Session 2026-01-02

- Q: CiliumはKube-proxy置換モードで動作していますか？ → A: Yes - kube-proxyは動作していない（Ciliumがkube-proxy置換モードで動作）
- Q: Helmチャートのバージョン管理方針は？ → A: 特定バージョンを固定（計画的アップグレード）
- Q: 初期段階で使用予定の主な機能は？ → A: Ingress - クラスター内サービスをtailnetに公開
- Q: Ingress検証用のテストサービスは？ → A: Longhorn Web UIを使用して検証（既存リソースを活用）

## Assumptions

- Tailscale OAuth client credentials have been created in the Tailscale admin console with appropriate scopes (Devices Core, Auth Keys, and Services write scopes)
- The OAuth client is configured with the `tag:k8s-operator` tag
- SOPS is configured in the cluster with age key for encryption/decryption
- Flux is already installed and operational in the cluster
- The cluster has network connectivity to Tailscale's coordination servers and Helm chart repository
- Kubernetes version is 1.23.0 or later (minimum supported by Tailscale Operator)
