# Quickstart: Deploy Actions Runner Controller (ARC)

**Date**: 2026-02-01  
**Feature**: 016-deploy-arc

## Prerequisites

1. Homelab Kubernetes cluster is running
2. FluxCD is operational
3. SOPS/Age encryption is configured
4. GitHub organization admin access

## Step 1: Create GitHub App

1. Go to your GitHub organization settings
2. Navigate to **Developer settings** → **GitHub Apps** → **New GitHub App**
3. Configure the app:
   - **GitHub App name**: `homelab-arc-runner`
   - **Homepage URL**: `https://github.com/actions/actions-runner-controller`
   - **Webhook**: Uncheck "Active" (not needed for scale sets)
4. Set permissions:
   - **Organization permissions**:
     - Self-hosted runners: **Read and write**
   - **Repository permissions**:
     - Metadata: **Read-only**
5. Click **Create GitHub App**
6. Note the **App ID** from the app settings page
7. Generate a **Private key** and download the `.pem` file
8. Click **Install App** and install it on your organization
9. Note the **Installation ID** from the URL: `https://github.com/organizations/<ORG>/settings/installations/<INSTALLATION_ID>`

## Step 2: Create SOPS-Encrypted Secret

```bash
# Navigate to the config directory
cd k8s/configs/arc-runners

# Create the secret file (will be encrypted)
cat > secret-github-app.sops.yaml << 'EOF'
apiVersion: v1
kind: Secret
metadata:
  name: arc-github-app
  namespace: arc-runners
type: Opaque
stringData:
  github_app_id: "YOUR_APP_ID"
  github_app_installation_id: "YOUR_INSTALLATION_ID"
  github_app_private_key: |
    -----BEGIN RSA PRIVATE KEY-----
    YOUR_PRIVATE_KEY_CONTENT
    -----END RSA PRIVATE KEY-----
EOF

# Encrypt with SOPS
sops -e -i secret-github-app.sops.yaml
```

## Step 3: Deploy via Git

```bash
# Commit and push
git add k8s/infrastructure/actions-runner-controller/
git add k8s/configs/arc-runners/
git commit -m "feat: deploy actions-runner-controller"
git push
```

FluxCD will automatically reconcile the changes.

## Step 4: Verify Deployment

```bash
# Check controller pods
kubectl get pods -n arc-systems

# Expected output:
# NAME                                              READY   STATUS    RESTARTS   AGE
# arc-controller-gha-runner-scale-set-controller-xxx   1/1     Running   0          2m

# Check listener pod
kubectl get pods -n arc-runners

# Expected output:
# NAME                              READY   STATUS    RESTARTS   AGE
# homelab-runners-xxxxx-listener    1/1     Running   0          1m
```

## Step 5: Test with a Workflow

Create a test workflow in any repository within your organization:

```yaml
# .github/workflows/test-arc.yaml
name: Test ARC Runner
on:
  workflow_dispatch:

jobs:
  test:
    runs-on: homelab
    steps:
      - name: Hello from homelab
        run: |
          echo "🎉 Running on homelab ARC runner!"
          echo "Hostname: $(hostname)"
          echo "CPU: $(nproc)"
          echo "Memory: $(free -h | grep Mem | awk '{print $2}')"
```

Trigger the workflow manually and verify:

```bash
# Watch runner pods scale up
kubectl get pods -n arc-runners -w

# Expected: A new runner pod appears when job starts
# NAME                                     READY   STATUS    RESTARTS   AGE
# homelab-runners-xxxxx-listener           1/1     Running   0          5m
# homelab-runners-xxxxx-runner-xxxxx       1/1     Running   0          10s
```

## Troubleshooting

### Controller not starting

```bash
# Check controller logs
kubectl logs -n arc-systems -l app.kubernetes.io/name=gha-runner-scale-set-controller

# Check HelmRelease status
kubectl get helmrelease -n arc-systems arc-controller -o yaml
```

### Listener not connecting to GitHub

```bash
# Check listener logs
kubectl logs -n arc-runners -l actions.github.com/scale-set-name=homelab

# Verify secret exists and has correct keys
kubectl get secret -n arc-runners arc-github-app -o yaml
```

### Runner pods not scaling

```bash
# Check AutoscalingRunnerSet status
kubectl get autoscalingrunnersets -n arc-runners

# Check listener is receiving jobs
kubectl logs -n arc-runners -l app.kubernetes.io/component=runner-scale-set-listener
```

## Configuration Reference

| Parameter | Value | Description |
|-----------|-------|-------------|
| Controller namespace | `arc-systems` | Where controller runs |
| Runner namespace | `arc-runners` | Where runner pods run |
| Runner label | `homelab` | Use in `runs-on:` |
| Min runners | 0 | Scale to zero when idle |
| Max runners | 3 | Maximum concurrent runners |
| CPU limit | 4 | Per runner pod |
| Memory limit | 8Gi | Per runner pod |
