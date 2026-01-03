# Implementation Plan: Grafana Alloy for Metrics and Logs

**Branch**: `004-grafana-alloy` | **Date**: 2026-01-03 | **Spec**: [spec.md](./spec.md)
**Input**: Feature specification from `/specs/004-grafana-alloy/spec.md`

## Summary

Deploy Grafana Alloy as a DaemonSet via GitOps (Flux) to collect Kubernetes cluster metrics and logs, forwarding them to Grafana Cloud. The implementation includes node metrics, pod metrics, Kubernetes state metrics (via integrated kube-state-metrics), and container logs with proper labeling (`cluster=homelab`). Credentials are stored encrypted with SOPS + age.

## Technical Context

**Language/Version**: YAML (Kubernetes manifests), Helm charts, River configuration (Grafana Alloy config language)
**Primary Dependencies**: Flux CD, Grafana Alloy Helm chart, SOPS + age
**Storage**: N/A (Alloy is stateless; data buffered in-memory during network issues)
**Testing**: Manual verification via Grafana Cloud dashboards, kubectl status checks, log queries in Loki
**Target Platform**: Kubernetes (Talos Linux single-node cluster)
**Project Type**: GitOps infrastructure configuration
**Performance Goals**: Metrics/logs visible in Grafana Cloud within 5 minutes (per spec SC-001, SC-004)
**Constraints**: Single-node cluster, no plaintext secrets in Git, all changes via Git/Flux reconciliation
**Scale/Scope**: Homelab cluster; monitoring all namespaces; single DaemonSet replica per node

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-check after Phase 1 design.*

| Principle | Status | Notes |
|-----------|--------|-------|
| I. GitOps-First | ✅ PASS | All resources deployed via Flux HelmRelease/Kustomization |
| II. Declarative Configuration | ✅ PASS | YAML manifests, Helm values, no imperative commands |
| III. Secrets Management | ✅ PASS | Grafana Cloud credentials encrypted with SOPS+age |
| IV. Immutable Infrastructure | ✅ PASS | No node-level changes required |
| V. Documentation as Code | ✅ PASS | Quickstart and operational docs included, Grafana Cloud setup documented |

**Post-Phase 1 Re-check**: All gates pass. No violations.

## Project Structure

### Documentation (this feature)

```text
specs/004-grafana-alloy/
├── plan.md              # This file
├── spec.md              # Feature specification
├── research.md          # Phase 0 output
├── data-model.md        # Phase 1 output
├── quickstart.md        # Phase 1 output
├── contracts/
│   └── reconciliation.md # Flux reconciliation contract
├── checklists/
│   └── requirements.md  # Spec quality checklist
└── tasks.md             # Phase 2 output (created by /speckit.tasks)
```

### Source Code (repository root)

```text
k8s/infrastructure/grafana-alloy/
├── kustomization.yaml              # Kustomize entry point
├── namespace.yaml                  # monitoring namespace
├── helmrepository.yaml             # Flux HelmRepository for Grafana charts
├── helmrelease.yaml                # Grafana Alloy HelmRelease with DaemonSet config
└── secret-grafana-cloud.sops.yaml  # SOPS-encrypted Grafana Cloud credentials

docs/
└── grafana-alloy.md                # Operational documentation including Grafana Cloud setup
```

**Structure Decision**: Follow existing infrastructure pattern (`k8s/infrastructure/<component>/`) with Kustomize + Helm. Deploy to `monitoring` namespace as decided in clarifications.

## Key Design Decisions

### 1. DaemonSet Deployment Mode

**Problem**: Need to collect logs from all nodes and ensure complete coverage.

**Solution**: Configure Grafana Alloy Helm chart with `controller.type: daemonset`.

**Rationale**: DaemonSet ensures one Alloy pod per node, which is required for complete log collection from all node's container runtime.

### 2. Alloy Configuration via Helm Values

**Problem**: Need to configure metrics scraping, log collection, and remote write endpoints.

**Solution**: Embed Alloy River configuration in HelmRelease values under `alloy.configMap.content`.

**Components configured**:
- `discovery.kubernetes` for pod/service discovery
- `prometheus.scrape` for node metrics (kubelet, cadvisor)
- `prometheus.operator.podmonitors` / `prometheus.operator.servicemonitors` (if needed)
- `loki.source.kubernetes` for container log collection
- `prometheus.remote_write` to Grafana Cloud Prometheus
- `loki.write` to Grafana Cloud Loki

### 3. Credential Injection via SOPS Secret

**Problem**: HelmRelease needs Grafana Cloud credentials (API token, endpoints) without exposing them in Git.

**Solution**: Create SOPS-encrypted Secret, use environment variables in Alloy config referencing the secret.

**Secret structure**:
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: grafana-cloud-credentials
  namespace: monitoring
type: Opaque
stringData:
  GRAFANA_CLOUD_PROMETHEUS_URL: "https://prometheus-prod-XX-XXX.grafana.net/api/prom/push"
  GRAFANA_CLOUD_LOKI_URL: "https://logs-prod-XX-XXX.grafana.net/loki/api/v1/push"
  GRAFANA_CLOUD_USER: "<user-id>"
  GRAFANA_CLOUD_API_KEY: "<api-key>"
```

### 4. Cluster Label for Multi-Cluster Identification

**Decision**: Add `cluster=homelab` label to all metrics and logs via Alloy's external_labels and log processing pipeline.

**Rationale**: Enables filtering in Grafana Cloud when multiple clusters report to the same tenant.

### 5. Metrics Collection Strategy

**Components**:
1. **Node metrics**: kubelet `/metrics` endpoint (CPU, memory, disk, network)
2. **Container metrics**: cadvisor metrics via kubelet `/metrics/cadvisor`
3. **Kubernetes state**: Built-in or deploy kube-state-metrics and scrape

**Note**: Grafana Alloy can integrate kube-state-metrics functionality. Evaluate during research if separate deployment is needed.

## Dependencies

| Dependency | Version | Purpose |
|------------|---------|---------|
| Grafana Alloy Helm Chart | Latest stable | Alloy deployment |
| Flux CD | Existing | GitOps reconciliation |
| SOPS | Existing | Secret encryption |
| Grafana Cloud Account | N/A | Metrics/logs backend |

## Risk Assessment

| Risk | Likelihood | Impact | Mitigation |
|------|------------|--------|------------|
| API token expiry | Low | Medium | Document renewal in quickstart |
| High log volume causing rate limits | Low | Medium | Configure log filtering if needed |
| Grafana Cloud endpoint changes | Low | Low | Pin endpoint URLs in secret |
| Alloy config syntax errors | Medium | Medium | Validate config before commit, check Alloy pod logs |

## Complexity Tracking

No constitution violations requiring justification.

## Phase Outputs

- [x] **Phase 0**: [research.md](./research.md) - Technical decisions documented
- [x] **Phase 1**: [data-model.md](./data-model.md) - Kubernetes resource definitions
- [x] **Phase 1**: [contracts/reconciliation.md](./contracts/reconciliation.md) - Flux reconciliation contract
- [x] **Phase 1**: [quickstart.md](./quickstart.md) - Deployment guide with Grafana Cloud setup
- [ ] **Phase 2**: [tasks.md](./tasks.md) - Task breakdown (created by /speckit.tasks)
