# Grafana Alloy for Metrics and Logs

**Feature**: 004-grafana-alloy  
**Date**: 2026-01-03

## Overview

Grafana Alloy is deployed as a DaemonSet to collect Kubernetes cluster metrics and logs, forwarding them to Grafana Cloud. This provides centralized observability for the homelab Kubernetes cluster.

## Architecture

- **Deployment**: DaemonSet (one pod per node)
- **Namespace**: `monitoring`
- **Metrics**: Collected from kubelet and cadvisor, sent to Grafana Cloud Prometheus
- **Logs**: Collected from all pods, sent to Grafana Cloud Loki
- **Cluster Label**: `cluster=homelab` added to all metrics and logs

## Prerequisites

- Kubernetes cluster running with Flux CD
- SOPS + age encryption configured
- Access to Grafana Cloud account

## Initial Setup

### Step 1: Create Grafana Cloud Access Policy Token

1. Log in to [Grafana Cloud](https://grafana.com)
2. Go to **My Account** → **Grafana Cloud** portal
3. Note your instance information:
   - **Instance ID** (numeric, e.g., `123456`) - username for auth
   - **Prometheus remote write URL** (e.g., `https://prometheus-prod-13-prod-us-east-0.grafana.net/api/prom/push`)
   - **Loki URL** (e.g., `https://logs-prod-006.grafana.net/loki/api/v1/push`)

4. Create Access Policy Token:
   - Go to **Security** → **Access Policies**
   - Click **Create access policy**
   - Configure:
     - **Name**: `homelab-alloy`
     - **Scopes**: Select `metrics:write` and `logs:write`
     - **Realms**: Select your stack
   - Click **Create**
   - Click **Add token**
   - **Copy the token immediately** - it won't be shown again

### Step 2: Create Encrypted Secret

1. Create unencrypted secret YAML (`secret-grafana-cloud.yaml`):

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: grafana-cloud-credentials
  namespace: monitoring
type: Opaque
stringData:
  GRAFANA_CLOUD_PROMETHEUS_URL: "https://prometheus-prod-XX-prod-XX-XXX.grafana.net/api/prom/push"
  GRAFANA_CLOUD_LOKI_URL: "https://logs-prod-XX.grafana.net/loki/api/v1/push"
  GRAFANA_CLOUD_USER: "YOUR_INSTANCE_ID"
  GRAFANA_CLOUD_API_KEY: "glc_YOUR_TOKEN_HERE"
```

2. Encrypt with SOPS:

```bash
cd /path/to/homelab-k8s
sops -e secret-grafana-cloud.yaml > k8s/infrastructure/grafana-alloy/secret-grafana-cloud.sops.yaml
rm secret-grafana-cloud.yaml
```

3. Verify encryption:

```bash
cat k8s/infrastructure/grafana-alloy/secret-grafana-cloud.sops.yaml
# Should show ENC[AES256_GCM,...] encrypted values
```

### Step 3: Deploy via GitOps

1. Commit changes:

```bash
git add k8s/infrastructure/grafana-alloy/
git commit -m "feat(monitoring): add Grafana Alloy for metrics and logs"
git push
```

2. Trigger reconciliation:

```bash
flux reconcile kustomization infrastructure --with-source
```

## Verification

### Check Deployment Status

```bash
# Check Flux Kustomization
flux get kustomizations infrastructure
# Expected: Ready True

# Check HelmRelease
flux get helmreleases -n monitoring
# Expected: Ready True

# Check DaemonSet
kubectl get daemonset -n monitoring
# Expected: DESIRED = READY = AVAILABLE

# Check pods
kubectl get pods -n monitoring -l app.kubernetes.io/name=alloy
# Expected: Running on each node
```

### Check Alloy Logs

```bash
# Check for successful remote write
kubectl logs -n monitoring -l app.kubernetes.io/name=alloy --tail=100 | grep -i "remote_write\|loki"
# Look for: "msg=sending" or similar success messages
```

### Verify in Grafana Cloud

**Metrics (Prometheus)**:
1. Go to Grafana Cloud → **Explore**
2. Select **Prometheus** data source
3. Run query: `up{cluster="homelab"}`
4. Expected: Results showing cluster nodes with value `1`

**Logs (Loki)**:
1. Go to Grafana Cloud → **Explore**
2. Select **Loki** data source
3. Run query: `{cluster="homelab"}`
4. Expected: Logs from cluster pods with namespace/pod/container labels

## Sample Queries

### Metrics (Prometheus)

```promql
# Node CPU usage
100 - (avg by (node) (rate(node_cpu_seconds_total{mode="idle",cluster="homelab"}[5m])) * 100)

# Pod memory usage
container_memory_working_set_bytes{cluster="homelab",container!=""}

# Kubernetes state
kube_pod_status_phase{cluster="homelab"}
```

### Logs (Loki)

```logql
# All logs from default namespace
{cluster="homelab",namespace="default"}

# Error logs only
{cluster="homelab"} |= "error"

# Logs from specific pod
{cluster="homelab",pod=~"nginx.*"}
```

## Troubleshooting

### No Metrics in Grafana Cloud

1. **Check Alloy logs for errors**:
   ```bash
   kubectl logs -n monitoring -l app.kubernetes.io/name=alloy | grep -i error
   ```

2. **Verify secret values**:
   ```bash
   kubectl get secret -n monitoring grafana-cloud-credentials -o jsonpath='{.data.GRAFANA_CLOUD_PROMETHEUS_URL}' | base64 -d
   ```

3. **Test connectivity**:
   ```bash
   kubectl run test --rm -it --restart=Never --image=curlimages/curl -- \
     curl -v https://prometheus-prod-XX-XXX.grafana.net/api/prom/push
   ```

### No Logs in Grafana Cloud

1. **Check log source configuration**:
   ```bash
   kubectl logs -n monitoring -l app.kubernetes.io/name=alloy | grep -i loki
   ```

2. **Verify pod discovery**:
   ```bash
   kubectl logs -n monitoring -l app.kubernetes.io/name=alloy | grep -i discovery
   ```

### Alloy Pod CrashLoopBackOff

1. **Check previous logs**:
   ```bash
   kubectl logs -n monitoring -l app.kubernetes.io/name=alloy --previous
   ```

2. **Common causes**:
   - Invalid River configuration syntax
   - Missing required environment variables
   - Insufficient RBAC permissions

### Authentication Errors

**Symptoms**: Logs show `401 Unauthorized` or `403 Forbidden`

**Resolution**:
1. Verify Access Policy Token has correct scopes (`metrics:write`, `logs:write`)
2. Check token hasn't expired
3. Verify instance ID matches Grafana Cloud account

## Credential Rotation

When rotating the Grafana Cloud API token:

1. Create new token in Grafana Cloud console
2. Update encrypted secret:
   ```bash
   sops k8s/infrastructure/grafana-alloy/secret-grafana-cloud.sops.yaml
   # Edit GRAFANA_CLOUD_API_KEY value
   # Save and exit
   ```
3. Commit and push:
   ```bash
   git add k8s/infrastructure/grafana-alloy/secret-grafana-cloud.sops.yaml
   git commit -m "chore(monitoring): rotate Grafana Cloud API token"
   git push
   ```
4. Flux will reconcile and restart Alloy pods with new credentials

## Configuration

### River Configuration

The Alloy configuration is embedded in the HelmRelease values under `alloy.configMap.content`. Key components:

- **Discovery**: `discovery.kubernetes` for nodes and pods
- **Metrics**: `prometheus.scrape` for kubelet and cadvisor metrics
- **Logs**: `loki.source.kubernetes` for pod logs
- **Remote Write**: `prometheus.remote_write` and `loki.write` to Grafana Cloud

### Environment Variables

Credentials are injected via environment variables from the Secret:
- `GRAFANA_CLOUD_PROMETHEUS_URL`
- `GRAFANA_CLOUD_LOKI_URL`
- `GRAFANA_CLOUD_USER`
- `GRAFANA_CLOUD_API_KEY`

## Resource Requirements

| Resource | Request | Limit | Notes |
|----------|---------|-------|-------|
| CPU | 100m | 500m | Per Alloy pod |
| Memory | 128Mi | 512Mi | Per Alloy pod; may need increase for high log volume |
| Storage | None | None | WAL stored in emptyDir |

## Next Steps

- [ ] Set up Grafana dashboards for Kubernetes monitoring
- [ ] Configure alerting rules in Grafana Cloud
- [ ] Consider adding additional metrics (custom application metrics)
- [ ] Tune log collection if volume is too high

## References

- [Grafana Alloy Documentation](https://grafana.com/docs/alloy/latest/)
- [Grafana Cloud Documentation](https://grafana.com/docs/grafana-cloud/)
- [Feature Specification](../specs/004-grafana-alloy/spec.md)
- [Quickstart Guide](../specs/004-grafana-alloy/quickstart.md)
