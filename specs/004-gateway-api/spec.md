# Feature Specification: Gateway API CRD Installation

**Feature Branch**: `004-gateway-api`
**Created**: 2026-01-03
**Status**: Draft
**Input**: User description: "Gateway APIの導入"

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Install Gateway API CRDs (Priority: P1)

As a cluster administrator, I want to install the Kubernetes Gateway API CRDs so that I can use Gateway and HTTPRoute resources to expose services via Tailscale Operator.

**Why this priority**: This is the foundational requirement. Without the Gateway API CRDs installed in the cluster, the Gateway and HTTPRoute resources defined in the Tailscale configuration cannot be created, and services cannot be exposed via the tailnet.

**Independent Test**: Can be fully tested by verifying that Gateway API CRDs are installed and that the existing Tailscale Gateway/HTTPRoute resources become functional.

**Acceptance Scenarios**:

1. **Given** a Kubernetes cluster without Gateway API CRDs, **When** the Gateway API CRDs are applied, **Then** the GatewayClass, Gateway, HTTPRoute, and related CRDs should be available in the cluster
2. **Given** Gateway API CRDs are installed, **When** running `kubectl get crd | grep gateway`, **Then** CRDs including `gatewayclasses.gateway.networking.k8s.io`, `gateways.gateway.networking.k8s.io`, and `httproutes.gateway.networking.k8s.io` should be listed
3. **Given** Gateway API CRDs are installed, **When** checking the existing Tailscale Gateway resource, **Then** the Gateway resource should transition to a valid state without API errors

---

### User Story 2 - Manage Gateway API via GitOps (Priority: P2)

As a cluster administrator, I want the Gateway API CRDs to be managed via Flux (GitOps) so that the configuration is version-controlled and automatically reconciled.

**Why this priority**: GitOps management ensures consistency, auditability, and automated deployment, aligning with the existing infrastructure management approach using Flux.

**Independent Test**: Can be tested by pushing changes to the repository and verifying Flux reconciles the Gateway API CRD resources automatically.

**Acceptance Scenarios**:

1. **Given** Flux is running in the cluster, **When** the Gateway API CRD manifest is committed to the repository, **Then** Flux should automatically install the Gateway API CRDs
2. **Given** Gateway API CRDs are managed by Flux, **When** checking the Flux Kustomization status, **Then** the Kustomization should show `Ready` status
3. **Given** Gateway API CRDs are managed by Flux, **When** the CRD version is updated in Git, **Then** Flux should reconcile the changes automatically

---

### User Story 3 - Enable Longhorn UI Access via Tailnet (Priority: P2)

As a cluster administrator, I want to verify that after installing Gateway API CRDs, the existing Tailscale Gateway and HTTPRoute configurations work correctly, allowing access to Longhorn UI via the tailnet.

**Why this priority**: This validates that the Gateway API installation is functional and integrates properly with the existing Tailscale Operator setup.

**Independent Test**: Can be tested by accessing the Longhorn UI via the tailnet hostname from any device connected to the tailnet.

**Acceptance Scenarios**:

1. **Given** Gateway API CRDs are installed and Tailscale Operator is running, **When** the Gateway resource is reconciled, **Then** a Tailscale device should be created for the gateway
2. **Given** the Tailscale Gateway is active, **When** accessing `longhorn.<tailnet>.ts.net` from a tailnet device, **Then** the Longhorn UI dashboard should be displayed
3. **Given** the HTTPRoute is configured, **When** checking the HTTPRoute status, **Then** the route should show accepted status with the parent gateway

---

### Edge Cases

- What happens when the Gateway API CRD version is incompatible with the Tailscale Operator version?
- How does the system handle CRD upgrade scenarios where existing resources need migration?
- What happens if the CRD installation is interrupted or partially applied?
- How does the system behave when multiple GatewayClasses are registered by different controllers?

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: System MUST install the Kubernetes Gateway API CRDs from the official kubernetes-sigs/gateway-api repository via remote URL reference in Flux Kustomization
- **FR-002**: System MUST use the standard channel CRDs (not experimental) for production stability
- **FR-003**: System MUST pin the Gateway API CRD version to v1.4.1 for reproducibility and planned upgrades
- **FR-004**: System MUST integrate with the existing Flux GitOps workflow for deployment and reconciliation
- **FR-005**: System MUST install CRDs as foundational infrastructure by placing them first in kustomization.yaml resources order (explicit `dependsOn` is unnecessary for cluster-level CRDs)
- **FR-006**: System MUST support the Gateway, GatewayClass, HTTPRoute, and ReferenceGrant CRDs at minimum
- **FR-007**: System MUST allow cross-namespace references for HTTPRoute to Gateway (via ReferenceGrant or Gateway configuration)

### Key Entities

- **CustomResourceDefinition (CRD)**: The Gateway API CRDs that define Gateway, GatewayClass, HTTPRoute, and other resources
- **Kustomization**: Flux Kustomization resource that manages the Gateway API CRD installation
- **Gateway**: Gateway API resource that defines the entry point for traffic (already exists in tailscale configuration)
- **HTTPRoute**: Gateway API resource that defines routing rules to backend services (already exists in tailscale configuration)
- **ReferenceGrant**: Gateway API resource that allows cross-namespace references (may be needed for HTTPRoute in longhorn-system to reference Gateway in tailscale namespace)

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: All Gateway API CRDs are installed and available within 2 minutes of Flux reconciliation
- **SC-002**: Flux Kustomization for Gateway API shows `Ready=True` condition after successful deployment
- **SC-003**: Existing Tailscale Gateway resource transitions to a programmed state within 5 minutes of CRD installation
- **SC-004**: Longhorn UI is accessible via tailnet hostname from any device connected to the tailnet
- **SC-005**: Gateway API CRDs can be upgraded by updating the version in the manifest without manual intervention
- **SC-006**: No API errors related to unknown resource types for Gateway or HTTPRoute after CRD installation

## Clarifications

### Session 2026-01-03

- Q: Gateway API CRDのバージョンは？ → A: v1.4.1（最新安定版）
- Q: CRDインストール方法は？ → A: リモートURL参照（GitHubリリースURLを直接Kustomizationで参照）
- Q: Flux依存関係の順序制御方法は？ → A: kustomization.yamlのresources順序で制御（CRDは基盤インフラとして最初に配置、明示的なdependsOnは不要）

## Assumptions

- Tailscale Kubernetes Operator is already installed and operational (from spec 003)
- Gateway and HTTPRoute resources for Tailscale are already defined but may be in error state due to missing CRDs
- Flux is already installed and operational in the cluster
- The cluster has network connectivity to GitHub for downloading CRD manifests
- Kubernetes version is compatible with Gateway API v1.4.x (requires Kubernetes 1.26+)
- The standard channel CRDs are sufficient for Tailscale Operator's Gateway API support
