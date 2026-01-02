# Feature Specification: Tailscale Kubernetes Operator with Ingress

**Feature Branch**: `003-tailscale-k8s-operator`
**Created**: 2026-01-03
**Status**: Draft
**Input**: User description: "tailscale operatorを導入し、クラスタ上のサービスをIngressでtailnet内に公開できるようにする。このとき、必要があればProxyGroupを使ってリソース使用量が節約できるか検討する。また、Ciliumをkube-proxyなしで使っているため、その点も注意する。"

## Clarifications

### Session 2026-01-03

- Q: ProxyGroupを使用する場合、レプリカ数をいくつに設定すべきか？ → A: 1レプリカ（リソース効率優先、シングルノードに最適）
- Q: Tailscale Ingressで公開されるサービスに付与するACLタグの設計は？ → A: 汎用タグ `tag:k8s` を全プロキシに付与（シンプルなACL管理）
- Q: Tailscale OAuthクライアントに必要なスコープは？ → A: 最小限スコープ: `Devices Core`, `Auth Keys`, `Services` (write)
- Q: スモークテストとしてどのサービスを公開するか？ → A: 既存のクラスタサービス（Longhorn UI）を公開
- Q: Tailscale Operatorをデプロイするnamespaceは？ → A: `tailscale` namespace（公式推奨のデフォルト）

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Expose Kubernetes services to tailnet via Ingress (Priority: P1)

As a cluster operator, I want to expose Kubernetes services to my tailnet using Tailscale Ingress so that I can securely access internal cluster services from any device on my tailnet without exposing them to the public internet.

**Why this priority**: This is the core functionality - securely exposing cluster services to the tailnet is the primary goal of introducing the Tailscale Kubernetes Operator.

**Independent Test**: After reconciliation, an existing cluster service (Longhorn UI) can be accessed from a device on the tailnet using its Tailscale hostname, with TLS automatically provisioned.

**Acceptance Scenarios**:

1. **Given** a Kubernetes service running in the cluster, **When** an Ingress resource with Tailscale annotations is created, **Then** the service becomes accessible from devices on the tailnet via its Tailscale hostname.
2. **Given** a service exposed via Tailscale Ingress, **When** a user accesses it from a tailnet device, **Then** the connection is secured with automatically provisioned TLS certificates.
3. **Given** a service exposed via Tailscale Ingress, **When** a user attempts to access it from outside the tailnet (public internet), **Then** the service is not accessible.

---

### User Story 2 - GitOps-managed Tailscale Operator deployment (Priority: P2)

As a cluster operator, I want the Tailscale Operator to be deployed and managed via GitOps (Flux) so that the operator configuration is version-controlled, reproducible, and follows the same deployment pattern as other cluster infrastructure.

**Why this priority**: GitOps management ensures consistency with existing infrastructure patterns and enables reproducible deployments.

**Independent Test**: After Flux reconciliation, the Tailscale Operator is running and registered as a device in the tailnet.

**Acceptance Scenarios**:

1. **Given** the GitOps repository contains Tailscale Operator manifests, **When** Flux reconciles the cluster state, **Then** the Tailscale Operator is deployed and running in the cluster.
2. **Given** the Tailscale Operator is deployed, **When** checking the Tailscale admin console, **Then** the operator appears as a registered device with the `tag:k8s-operator` tag.
3. **Given** the Tailscale OAuth credentials are stored encrypted in Git, **When** Flux reconciles, **Then** the credentials are decrypted and available to the operator without plaintext secrets in Git.

---

### User Story 3 - Resource-efficient proxy management with ProxyGroup (Priority: P3)

As a cluster operator, I want to optionally use ProxyGroup for managing Tailscale proxies so that I can reduce resource consumption when exposing multiple services, while maintaining high availability through multiple replicas.

**Why this priority**: Resource efficiency is important for homelab environments, and ProxyGroup provides a way to share proxy infrastructure across multiple services while also enabling high availability.

**Independent Test**: Multiple services can be exposed through a single ProxyGroup, reducing the total number of proxy pods compared to per-service proxies.

**Acceptance Scenarios**:

1. **Given** a ProxyGroup is configured with 1 replica (optimized for single-node cluster), **When** services are exposed using the ProxyGroup, **Then** all services share the same proxy infrastructure instead of creating individual proxy pods.
3. **Given** a ProxyGroup is configured, **When** multiple Ingress resources reference it, **Then** the total resource consumption is lower than creating individual proxies per service.

---

### User Story 4 - Cilium kube-proxy replacement compatibility (Priority: P1)

As a cluster operator, I want the Tailscale Operator to work correctly with Cilium running in kube-proxy replacement mode so that Tailscale Ingress functions properly without conflicts with the CNI configuration.

**Why this priority**: The cluster uses Cilium with `kubeProxyReplacement: true`, which requires specific configuration to ensure Tailscale proxies can correctly route traffic to ClusterIP services.

**Independent Test**: Services exposed via Tailscale Ingress are accessible and correctly route traffic to backend pods despite Cilium's socket-level load balancing.

**Acceptance Scenarios**:

1. **Given** Cilium is configured with kube-proxy replacement mode, **When** a Tailscale Ingress exposes a ClusterIP service, **Then** traffic is correctly routed to the service's backend pods.
2. **Given** the Tailscale Operator is deployed with Cilium compatibility settings, **When** checking proxy pod connectivity to cluster services, **Then** connections succeed without being intercepted by Cilium's socket load balancer.

---

### Edge Cases

- What happens when the Tailscale OAuth token expires or is revoked?
- What happens when the tailnet has ACL rules that block the operator or exposed services?
- How does the system behave when a ProxyGroup replica fails during a request?
- What happens when Cilium's socket load balancing intercepts traffic intended for Tailscale proxies?
- How does the system handle certificate renewal for Tailscale Ingress services?
- What happens when the cluster has network policies that block Tailscale proxy traffic?

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: The system MUST deploy the Tailscale Kubernetes Operator via GitOps (Flux) workflow in the `tailscale` namespace.
- **FR-002**: The system MUST support exposing Kubernetes services to the tailnet using Tailscale Ingress resources.
- **FR-003**: The system MUST automatically provision TLS certificates for services exposed via Tailscale Ingress.
- **FR-004**: The system MUST store Tailscale OAuth credentials encrypted in Git using SOPS with age.
- **FR-005**: The system MUST decrypt OAuth credentials during Flux reconciliation.
- **FR-006**: The system MUST register the Tailscale Operator as a device in the tailnet with the `tag:k8s-operator` tag, and proxy devices with the `tag:k8s` tag for simplified ACL management.
- **FR-007**: The system MUST configure Cilium to bypass socket load balancing for Tailscale proxy pods to ensure compatibility with kube-proxy replacement mode.
- **FR-008**: The system SHOULD support ProxyGroup for resource-efficient proxy management when exposing multiple services, configured with 1 replica for the single-node cluster.
- **FR-009**: The system MUST ensure services exposed via Tailscale Ingress are only accessible from within the tailnet.
- **FR-010**: The system MUST provide operator-visible signals (events/status) for Ingress provisioning success/failure.

### Key Entities *(include if feature involves data)*

- **Tailscale Operator**: The Kubernetes operator that manages Tailscale resources and proxies in the cluster.
- **Tailscale Ingress**: A Kubernetes Ingress resource annotated to be handled by Tailscale, exposing a service to the tailnet.
- **ProxyGroup**: A shared proxy infrastructure that can serve multiple Ingress/egress configurations with configurable replicas.
- **OAuth Credential**: Authentication material for the Tailscale API with minimum required scopes (`Devices Core`, `Auth Keys`, `Services` write), stored encrypted in Git and used by the operator.
- **Tailnet Device**: A registered device in the Tailscale network, including the operator and any proxy pods.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: The Longhorn UI service can be exposed via Tailscale Ingress and accessed from a tailnet device within 5 minutes of creating the Ingress resource.
- **SC-002**: TLS certificates are automatically provisioned for exposed services, with HTTPS connections succeeding on first access.
- **SC-003**: The Git repository contains 0 plaintext instances of Tailscale OAuth credentials (as verified by keyword search) at all times.
- **SC-004**: Services exposed via Tailscale Ingress return connection refused or timeout when accessed from outside the tailnet (public internet).
- **SC-005**: The Tailscale Operator appears in the Tailscale admin console with the correct tags within 5 minutes of deployment.
- **SC-006**: When using ProxyGroup, exposing N services results in a fixed number of proxy pods (equal to ProxyGroup replicas) rather than N individual proxy pods.
- **SC-007**: Traffic to services exposed via Tailscale Ingress is correctly routed despite Cilium running in kube-proxy replacement mode.
