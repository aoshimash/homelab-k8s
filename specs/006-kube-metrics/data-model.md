# Data Model: kube-state-metrics

**Feature Branch**: `006-kube-metrics`  
**Date**: 2026-01-03

## Overview

This document describes the data entities involved in the kube-state-metrics implementation. Unlike typical application data models, this feature deals with Kubernetes resources and Prometheus metrics.

---

## Kubernetes Resources

### 1. HelmRepository

Defines the source for the kube-state-metrics Helm chart.

| Field | Value | Purpose |
|-------|-------|---------|
| `metadata.name` | `prometheus-community` | Identifier for Flux |
| `metadata.namespace` | `flux-system` | Standard Flux resource location |
| `spec.url` | `https://prometheus-community.github.io/helm-charts` | Chart repository URL |
| `spec.interval` | `1h0m0s` | How often to check for updates |

### 2. HelmRelease

Defines the kube-state-metrics deployment.

| Field | Value | Purpose |
|-------|-------|---------|
| `metadata.name` | `kube-state-metrics` | Release identifier |
| `metadata.namespace` | `monitoring` | Target namespace |
| `spec.chart.spec.chart` | `kube-state-metrics` | Chart name |
| `spec.chart.spec.sourceRef` | `prometheus-community` | Reference to HelmRepository |
| `spec.releaseName` | `kube-state-metrics` | Helm release name |
| `spec.values` | (optional overrides) | Configuration values |

### 3. Grafana Alloy ConfigMap (Update)

Additional configuration block for kube-state-metrics scraping.

| Component | Type | Purpose |
|-----------|------|---------|
| `discovery.kubernetes "kube_state_metrics"` | Discovery | Find kube-state-metrics service |
| `prometheus.scrape "kube_state_metrics"` | Scrape | Collect metrics from endpoint |

---

## Prometheus Metrics (Output)

kube-state-metrics generates the following metric families. These are stored in Grafana Cloud Prometheus.

### Core Metric Families

| Metric Prefix | Kubernetes Resource | Example Metrics |
|---------------|---------------------|-----------------|
| `kube_deployment_*` | Deployment | `spec_replicas`, `status_replicas_available`, `status_replicas_updated` |
| `kube_statefulset_*` | StatefulSet | `spec_replicas`, `status_replicas_ready` |
| `kube_daemonset_*` | DaemonSet | `status_desired_number_scheduled`, `status_number_ready` |
| `kube_replicaset_*` | ReplicaSet | `spec_replicas`, `status_ready_replicas` |
| `kube_pod_*` | Pod | `status_phase`, `container_status_ready`, `container_resource_limits` |
| `kube_node_*` | Node | `status_condition`, `status_capacity`, `info` |
| `kube_service_*` | Service | `info`, `spec_type` |
| `kube_pv_*` | PersistentVolume | `status_phase`, `capacity_bytes` |
| `kube_pvc_*` | PersistentVolumeClaim | `status_phase`, `resource_requests_storage_bytes` |
| `kube_namespace_*` | Namespace | `status_phase`, `labels` |
| `kube_job_*` | Job | `status_succeeded`, `status_failed`, `complete` |
| `kube_cronjob_*` | CronJob | `status_active`, `next_schedule_time` |

### Common Labels

All metrics include these labels for filtering:

| Label | Source | Example |
|-------|--------|---------|
| `namespace` | Kubernetes metadata | `monitoring`, `default` |
| `pod` | Kubernetes metadata | `nginx-7c9b8d6f4-abc12` |
| `node` | Kubernetes metadata | `talos-controlplane-1` |
| `cluster` | Added by Grafana Alloy relabeling | `homelab` |
| `job` | Scrape job name | `kube-state-metrics` |

---

## Data Flow

```text
┌─────────────────────┐
│  Kubernetes API     │
│  (Deployments,      │
│   Pods, Nodes...)   │
└─────────┬───────────┘
          │ Watch/List
          ▼
┌─────────────────────┐
│  kube-state-metrics │
│  (Deployment in     │
│   monitoring ns)    │
└─────────┬───────────┘
          │ /metrics endpoint
          ▼
┌─────────────────────┐
│  Grafana Alloy      │
│  (DaemonSet in      │
│   monitoring ns)    │
│  - Scrape metrics   │
│  - Add cluster label│
└─────────┬───────────┘
          │ Remote write
          ▼
┌─────────────────────┐
│  Grafana Cloud      │
│  Prometheus         │
└─────────────────────┘
```

---

## State Transitions

### kube-state-metrics Pod Lifecycle

| State | Trigger | Next State |
|-------|---------|------------|
| Pending | HelmRelease created | Running |
| Running | Normal operation | Running |
| Running | Pod eviction/OOM | Pending (rescheduled) |
| Running | HelmRelease deleted | Terminated |

### Metric Data Lifecycle

| Stage | Retention | Notes |
|-------|-----------|-------|
| kube-state-metrics buffer | ~0 (stateless) | No local storage |
| Grafana Alloy buffer | Configurable (default ~5min) | Handles temporary Grafana Cloud outage |
| Grafana Cloud Prometheus | Per plan (typically 15-30 days) | Managed by Grafana Cloud |
