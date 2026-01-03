# Feature Specification: Grafana Alloy for Metrics and Logs to Grafana Cloud

**Feature Branch**: `004-grafana-alloy`
**Created**: 2026-01-03
**Status**: Draft
**Input**: User description: "Grafana Alloyを使ってGrafanaCloudにMetricsとLogsを送る"

## Clarifications

### Session 2026-01-03

- Q: Grafana Alloyのデプロイモードは？ → A: DaemonSet（各ノードに1 Pod、ログ収集に最適）
- Q: 収集対象のnamespaceは？ → A: 全namespace（クラスタ全体を監視）
- Q: Grafana Alloyをデプロイするnamespaceは？ → A: `monitoring`（観測性ツールの標準namespace）
- Q: Grafana Cloud認証方式は？ → A: API Token（Grafana Cloud Access Policy Token）。Grafana Cloud側の設定手順はドキュメントに記載する。
- Q: クラスタ識別ラベルの値は？ → A: `homelab`

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Send cluster metrics to Grafana Cloud (Priority: P1)

As a cluster operator, I want Grafana Alloy to collect and send Kubernetes cluster metrics to Grafana Cloud so that I can monitor cluster health, resource usage, and workload performance from a centralized observability platform.

**Why this priority**: Metrics are foundational for observability - understanding cluster health, resource consumption, and identifying issues requires reliable metrics collection and visualization.

**Independent Test**: After reconciliation, Grafana Cloud dashboards display real-time metrics from the Kubernetes cluster including node metrics, pod metrics, and Kubernetes state metrics.

**Acceptance Scenarios**:

1. **Given** Grafana Alloy is deployed and configured, **When** checking Grafana Cloud, **Then** Kubernetes node metrics (CPU, memory, disk, network) are visible and updating.
2. **Given** Grafana Alloy is collecting metrics, **When** a workload is deployed to the cluster, **Then** pod-level metrics for the workload appear in Grafana Cloud within 5 minutes.
3. **Given** Grafana Alloy is running, **When** querying Prometheus-compatible endpoints in Grafana Cloud, **Then** Kubernetes state metrics (deployments, pods, services) are available.

---

### User Story 2 - Send cluster logs to Grafana Cloud (Priority: P1)

As a cluster operator, I want Grafana Alloy to collect and send Kubernetes pod logs to Grafana Cloud so that I can search, analyze, and troubleshoot application issues from a centralized logging platform.

**Why this priority**: Logs are essential for debugging and troubleshooting - having centralized logs alongside metrics enables faster incident response and root cause analysis.

**Independent Test**: After reconciliation, Grafana Cloud Logs (Loki) contains searchable logs from cluster workloads with proper labels for filtering by namespace, pod, and container.

**Acceptance Scenarios**:

1. **Given** Grafana Alloy is deployed and configured, **When** a pod writes to stdout/stderr, **Then** those logs appear in Grafana Cloud Logs (Loki) within 5 minutes.
2. **Given** logs are being collected, **When** searching in Grafana Cloud by namespace or pod name, **Then** logs can be filtered and retrieved accurately.
3. **Given** a multi-container pod, **When** viewing logs in Grafana Cloud, **Then** logs from each container are distinguishable via container labels.

---

### User Story 3 - GitOps-managed Grafana Alloy deployment (Priority: P2)

As a cluster operator, I want Grafana Alloy to be deployed and managed via GitOps (Flux) so that the observability configuration is version-controlled, reproducible, and follows the same deployment pattern as other cluster infrastructure.

**Why this priority**: GitOps management ensures consistency with existing infrastructure patterns (Longhorn, Tailscale Operator) and enables reproducible deployments.

**Independent Test**: After Flux reconciliation, Grafana Alloy is running and sending data to Grafana Cloud without manual intervention.

**Acceptance Scenarios**:

1. **Given** the GitOps repository contains Grafana Alloy manifests, **When** Flux reconciles the cluster state, **Then** Grafana Alloy is deployed and running in the cluster.
2. **Given** Grafana Cloud credentials are stored encrypted in Git, **When** Flux reconciles, **Then** the credentials are decrypted and available to Alloy without plaintext secrets in Git.
3. **Given** a configuration change is committed to Git, **When** Flux reconciles, **Then** Grafana Alloy picks up the new configuration automatically.

---

### Edge Cases

- What happens when Grafana Cloud is temporarily unreachable or returns authentication errors?
- How does the system behave when Grafana Cloud rate limits are exceeded?
- What happens when the cluster has a large volume of logs that could overwhelm network bandwidth?
- How does the system handle log collection from short-lived pods that terminate quickly?
- What happens when encryption keys for credential decryption are missing or rotated during reconciliation?
- How does the system behave when Grafana Alloy pods are restarted during metrics/logs collection?

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: The system MUST deploy Grafana Alloy as a DaemonSet in the `monitoring` namespace via GitOps (Flux) workflow, ensuring one pod runs on each node for complete log collection.
- **FR-002**: The system MUST collect and send Kubernetes node metrics (CPU, memory, disk, network) to Grafana Cloud Prometheus.
- **FR-003**: The system MUST collect and send Kubernetes pod/container metrics from all namespaces to Grafana Cloud Prometheus.
- **FR-004**: The system MUST collect and send Kubernetes state metrics (deployments, pods, services, etc.) to Grafana Cloud Prometheus.
- **FR-005**: The system MUST collect and send pod logs (stdout/stderr) from all namespaces to Grafana Cloud Loki.
- **FR-006**: The system MUST add appropriate labels to metrics and logs including cluster identifier (`cluster=homelab`), namespace, pod, and container for filtering and searching.
- **FR-007**: The system MUST authenticate to Grafana Cloud using API Token (Access Policy Token) for metrics and logs push.
- **FR-008**: The system MUST store Grafana Cloud credentials (API Token, endpoint URLs) encrypted in Git using SOPS with age.
- **FR-009**: The system MUST decrypt credentials during Flux reconciliation.
- **FR-010**: The documentation MUST include step-by-step instructions for creating Grafana Cloud Access Policy Token with required scopes (metrics:write, logs:write).
- **FR-011**: The system MUST handle temporary Grafana Cloud connectivity issues gracefully with appropriate buffering or retry mechanisms.
- **FR-012**: The system SHOULD provide operator-visible signals (events/status) for data collection and transmission success/failure.

### Key Entities *(include if feature involves data)*

- **Grafana Alloy**: The observability collector agent that gathers metrics and logs from the cluster and ships them to Grafana Cloud.
- **Grafana Cloud Prometheus**: The remote metrics backend where cluster metrics are stored and queried.
- **Grafana Cloud Loki**: The remote logging backend where cluster logs are stored and searched.
- **Grafana Cloud Credential**: Authentication material including Access Policy Token (with metrics:write, logs:write scopes), Prometheus remote write endpoint URL, and Loki push endpoint URL, stored encrypted in Git.
- **Metrics**: Time-series data representing cluster and workload resource usage and state.
- **Logs**: Textual output from containers (stdout/stderr) with associated metadata labels.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: Kubernetes node metrics (CPU, memory, disk) appear in Grafana Cloud within 5 minutes of Grafana Alloy deployment.
- **SC-002**: Pod-level metrics for newly deployed workloads appear in Grafana Cloud within 5 minutes of pod creation.
- **SC-003**: Kubernetes state metrics (kube-state-metrics equivalent) are queryable in Grafana Cloud Prometheus.
- **SC-004**: Pod logs appear in Grafana Cloud Loki within 5 minutes of log generation.
- **SC-005**: Logs and metrics can be filtered by cluster (`homelab`), namespace, pod name, and container name with 100% accuracy in label matching.
- **SC-006**: The Git repository contains 0 plaintext instances of Grafana Cloud credentials (API tokens, passwords) at all times.
- **SC-007**: Grafana Alloy continues operating and buffers data during temporary Grafana Cloud connectivity issues (simulated by network interruption), resuming transmission when connectivity is restored.
