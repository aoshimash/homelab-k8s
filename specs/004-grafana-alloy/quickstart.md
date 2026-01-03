# Quickstart: Grafana Alloy for Metrics and Logs

**Feature**: 004-grafana-alloy
**Date**: 2026-01-03

## Prerequisites

- Kubernetes cluster running with Flux CD
- SOPS + age encryption configured
- Access to Grafana Cloud account

## Step 1: Create Grafana Cloud Access Policy Token

### 1.1 Log in to Grafana Cloud

Navigate to [grafana.com](https://grafana.com) and log in to your account.

### 1.2 Get Your Instance Information

1. Go to **My Account** → **Grafana Cloud** portal
2. Find your stack and note:
   - **Instance ID** (numeric, e.g., `123456`) - this is your username for auth
   - **Prometheus remote write URL** (e.g., `https://prometheus-prod-13-prod-us-east-0.grafana.net/api/prom/push`)
   - **Loki URL** (e.g., `https://logs-prod-006.grafana.net/loki/api/v1/push`)

### 1.3 Create Access Policy Token

1. Go to **Security** → **Access Policies**
2. Click **Create access policy**
3. Configure:
   - **Name**: `homelab-alloy`
   - **Scopes**: Select `metrics:write` and `logs:write`
   - **Realms**: Select your stack
4. Click **Create**
5. Click **Add token**
6. **Copy the token immediately** - it won't be shown again

## Step 2: Create Encrypted Secret

### 2.1 Create Secret YAML

Create `secret-grafana-cloud.yaml` (unencrypted, temporary):

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

Replace placeholders with your actual values from Step 1.

### 2.2 Encrypt with SOPS

```bash
# Navigate to repository root
cd /path/to/homelab-k8s

# Encrypt the secret
sops -e secret-grafana-cloud.yaml > k8s/infrastructure/grafana-alloy/secret-grafana-cloud.sops.yaml

# Remove the unencrypted file
rm secret-grafana-cloud.yaml

# Verify encryption
cat k8s/infrastructure/grafana-alloy/secret-grafana-cloud.sops.yaml
# Should show ENC[AES256_GCM,...] encrypted values
```

## Step 3: Deploy via GitOps

### 3.1 Commit Changes

```bash
git add k8s/infrastructure/grafana-alloy/
git commit -m "feat(monitoring): add Grafana Alloy for metrics and logs

Deploy Grafana Alloy as DaemonSet to collect cluster metrics and logs,
forwarding to Grafana Cloud. Includes SOPS-encrypted credentials."
git push
```

### 3.2 Trigger Reconciliation (or wait)

```bash
# Force immediate reconciliation
flux reconcile kustomization infrastructure --with-source

# Or wait for automatic reconciliation (10 minutes)
```

## Step 4: Verify Deployment

### 4.1 Check Flux Status

```bash
# Check kustomization
flux get kustomizations infrastructure
# Expected: Ready True

# Check HelmRelease
flux get helmreleases -n monitoring
# Expected: Ready True
```

### 4.2 Check Pods

```bash
# Check DaemonSet
kubectl get daemonset -n monitoring
# Expected: DESIRED = READY = AVAILABLE

# Check pods are running
kubectl get pods -n monitoring -l app.kubernetes.io/name=alloy
# Expected: Running on each node
```

### 4.3 Check Logs

```bash
# Check for successful remote write
kubectl logs -n monitoring -l app.kubernetes.io/name=alloy --tail=100 | grep -i "remote_write\|loki"
# Look for: "msg=sending" or similar success messages
# Errors: "401 Unauthorized" = bad credentials, "connection refused" = network issue
```

## Step 5: Verify in Grafana Cloud

### 5.1 Check Metrics

1. Go to Grafana Cloud → **Explore**
2. Select **Prometheus** data source
3. Run query: `up{cluster="homelab"}`
4. Expected: Results showing cluster nodes with value `1`

### 5.2 Check Logs

1. Go to Grafana Cloud → **Explore**
2. Select **Loki** data source
3. Run query: `{cluster="homelab"}`
4. Expected: Logs from cluster pods with namespace/pod/container labels

### 5.3 Sample Queries

**Metrics (Prometheus)**:
```promql
# Node CPU usage
100 - (avg by (node) (rate(node_cpu_seconds_total{mode="idle",cluster="homelab"}[5m])) * 100)

# Pod memory usage
container_memory_working_set_bytes{cluster="homelab",container!=""}

# Kubernetes state
kube_pod_status_phase{cluster="homelab"}
```

**Logs (Loki)**:
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

### Credential Rotation

When you need to rotate the Grafana Cloud API token:

1. Create new token in Grafana Cloud console
2. Update `secret-grafana-cloud.sops.yaml`:
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

## Next Steps

- [ ] Set up Grafana dashboards for Kubernetes monitoring
- [ ] Configure alerting rules in Grafana Cloud
- [ ] Consider adding additional metrics (custom application metrics)
- [ ] Tune log collection if volume is too high
