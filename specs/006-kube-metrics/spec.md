# Feature Specification: kube-state-metrics for Kubernetes Resource State Monitoring

**Feature Branch**: `006-kube-metrics`  
**Created**: 2026-01-03  
**Status**: Draft  
**Input**: User description: "kube-metrics導入"

## Overview

This feature introduces kube-state-metrics to the homelab Kubernetes cluster. kube-state-metrics is a service that listens to the Kubernetes API server and generates metrics about the state of Kubernetes objects (deployments, pods, nodes, services, etc.). These metrics complement the existing kubelet and cAdvisor metrics already collected by Grafana Alloy, providing a complete observability picture of the cluster.

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Monitor Kubernetes Resource States (Priority: P1)

As a cluster operator, I want to see the state of Kubernetes resources (Deployments, Pods, ReplicaSets, Services, etc.) as metrics in Grafana Cloud so that I can understand the health and status of my workloads beyond just CPU and memory usage.

**Why this priority**: kube-state-metrics provides essential visibility into Kubernetes object states that kubelet and cAdvisor cannot provide. Without this, operators cannot see metrics like "number of desired replicas vs actual replicas", "pod phase", or "deployment rollout status" which are critical for understanding cluster health.

**Independent Test**: After deployment, Grafana Cloud displays metrics such as `kube_deployment_spec_replicas`, `kube_pod_status_phase`, and `kube_node_status_condition` with accurate values reflecting the cluster state.

**Acceptance Scenarios**:

1. **Given** kube-state-metrics is deployed and Grafana Alloy is scraping it, **When** checking Grafana Cloud, **Then** deployment metrics (desired replicas, available replicas, updated replicas) are visible and accurate.
2. **Given** kube-state-metrics is running, **When** a pod enters a non-Running state (Pending, Failed, etc.), **Then** the `kube_pod_status_phase` metric reflects this state within 2 minutes.
3. **Given** kube-state-metrics is collecting node metrics, **When** a node condition changes (e.g., memory pressure), **Then** the `kube_node_status_condition` metric updates accordingly.

---

### User Story 2 - GitOps-managed kube-state-metrics Deployment (Priority: P1)

As a cluster operator, I want kube-state-metrics to be deployed via GitOps (Flux) following the same patterns as other infrastructure components so that the deployment is reproducible, version-controlled, and consistent with the existing cluster management approach.

**Why this priority**: Consistency with existing GitOps patterns (Grafana Alloy, Longhorn, Tailscale Operator) ensures maintainability and reduces operational complexity.

**Independent Test**: After committing the kube-state-metrics manifests to Git, Flux reconciles and the deployment runs without manual intervention.

**Acceptance Scenarios**:

1. **Given** kube-state-metrics manifests exist in the Git repository, **When** Flux reconciles, **Then** kube-state-metrics is deployed to the cluster in the `monitoring` namespace.
2. **Given** a configuration change is committed to Git, **When** Flux reconciles, **Then** kube-state-metrics picks up the new configuration automatically.
3. **Given** the HelmRelease is configured, **When** checking the cluster, **Then** kube-state-metrics runs as a single-replica Deployment (not DaemonSet).

---

### User Story 3 - Integrate kube-state-metrics with Grafana Alloy (Priority: P1)

As a cluster operator, I want Grafana Alloy to automatically scrape metrics from kube-state-metrics and send them to Grafana Cloud so that Kubernetes state metrics are available alongside node and container metrics in the same observability platform.

**Why this priority**: Integration with the existing metrics pipeline ensures all metrics flow to Grafana Cloud with consistent labeling (cluster=homelab) and without requiring a separate metrics collection system.

**Independent Test**: kube-state-metrics are queryable in Grafana Cloud Prometheus with the `cluster=homelab` label, alongside existing kubelet and cAdvisor metrics.

**Acceptance Scenarios**:

1. **Given** kube-state-metrics is deployed and Grafana Alloy is configured to scrape it, **When** querying Grafana Cloud, **Then** kube-state-metrics appear with the `cluster=homelab` label.
2. **Given** Grafana Alloy is scraping kube-state-metrics, **When** new Kubernetes resources are created, **Then** metrics for those resources appear in Grafana Cloud within 5 minutes.
3. **Given** the metrics pipeline is working, **When** viewing metrics in Grafana Cloud, **Then** metrics from kubelet, cAdvisor, and kube-state-metrics are all available and correlatable.

---

### Edge Cases

- What happens when kube-state-metrics pod is unavailable? (Grafana Alloy should handle scrape failures gracefully)
- How does the system behave when the Kubernetes API server is temporarily overloaded?
- What happens during cluster upgrades when Kubernetes API versions change?
- How are metrics handled for short-lived resources (e.g., Jobs, completed Pods)?

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: The system MUST deploy kube-state-metrics via Flux HelmRelease in the `monitoring` namespace.
- **FR-002**: kube-state-metrics MUST run as a single-replica Deployment (not DaemonSet) since it only needs to query the Kubernetes API once.
- **FR-003**: kube-state-metrics MUST expose metrics for core Kubernetes resources: Deployments, StatefulSets, DaemonSets, ReplicaSets, Pods, Nodes, Services, PersistentVolumes, PersistentVolumeClaims, Namespaces, Jobs, and CronJobs.
- **FR-004**: Grafana Alloy MUST scrape metrics from kube-state-metrics Service endpoint.
- **FR-005**: Grafana Alloy MUST add the `cluster=homelab` label to all kube-state-metrics before sending to Grafana Cloud.
- **FR-006**: The system MUST follow the existing GitOps patterns: HelmRelease, HelmRepository, namespace.yaml, kustomization.yaml structure.
- **FR-007**: kube-state-metrics SHOULD have appropriate resource requests/limits to prevent resource contention.
- **FR-008**: The system SHOULD use Kubernetes service discovery to automatically find the kube-state-metrics endpoint.

### Key Entities

- **kube-state-metrics**: A Kubernetes service that generates metrics about the state of Kubernetes objects by listening to the Kubernetes API server.
- **Kubernetes State Metrics**: Time-series metrics representing the state of Kubernetes resources (e.g., `kube_deployment_spec_replicas`, `kube_pod_status_phase`, `kube_node_status_condition`).
- **Grafana Alloy**: The existing observability collector that will be extended to scrape kube-state-metrics.
- **Grafana Cloud Prometheus**: The remote metrics backend where kube-state-metrics will be stored and queried.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: kube-state-metrics pods are running and healthy in the `monitoring` namespace after Flux reconciliation.
- **SC-002**: Metrics with prefix `kube_` (e.g., `kube_deployment_spec_replicas`, `kube_pod_status_phase`) are queryable in Grafana Cloud within 10 minutes of deployment.
- **SC-003**: All kube-state-metrics in Grafana Cloud have the `cluster=homelab` label.
- **SC-004**: Kubernetes resource state changes (e.g., scaling a deployment) are reflected in Grafana Cloud metrics within 5 minutes.
- **SC-005**: The Git repository contains all required manifests following the existing structure pattern (no manual kubectl commands required for deployment).
- **SC-006**: kube-state-metrics memory usage remains stable and does not exceed reasonable limits for a small homelab cluster.
