# Actions Runner Controller (ARC) Deployment

This directory contains the Kubernetes manifests for deploying Actions Runner Controller to the homelab cluster.

## Status: Ready for Deployment

✅ Phase 1-3 completed: All manifests created and committed.

## Next Steps (Manual)

### Before Deployment

1. **Create GitHub App** (T004-T006):
   - Go to your GitHub organization settings
   - Navigate to Developer settings → GitHub Apps → New GitHub App
   - Configure:
     - Name: `homelab-arc-runner`
     - Homepage URL: `https://github.com/actions/actions-runner-controller`
     - Webhook: Uncheck "Active"
   - Set permissions:
     - Organization permissions → Self-hosted runners: Read and write
     - Repository permissions → Metadata: Read-only
   - Create the app and note the **App ID**
   - Generate and download **Private key** (.pem file)
   - Install app to your organization and note **Installation ID** from URL

2. **Update Secret** (T009):
   ```bash
   # Edit the secret file
   vi k8s/configs/arc-runners/secret-github-app.sops.yaml
   
   # Replace placeholders:
   # - REPLACE_WITH_YOUR_APP_ID
   # - REPLACE_WITH_YOUR_INSTALLATION_ID
   # - REPLACE_WITH_YOUR_PRIVATE_KEY_CONTENT
   
   # Encrypt with SOPS
   sops -e -i k8s/configs/arc-runners/secret-github-app.sops.yaml
   ```

3. **Update GitHub Organization URL**:
   ```bash
   # Edit the runner HelmRelease
   vi k8s/configs/arc-runners/helmrelease.yaml
   
   # Replace REPLACE_WITH_YOUR_ORG with your actual organization name
   ```

### Deployment (T016)

```bash
# Commit and push to trigger FluxCD reconciliation
git add k8s/configs/arc-runners/secret-github-app.sops.yaml
git add k8s/configs/arc-runners/helmrelease.yaml
git commit -m "feat(arc): configure GitHub App credentials and organization"
git push
```

### Verification (T017-T020)

```bash
# T017: Verify controller pod
kubectl get pods -n arc-systems
# Expected: arc-controller-* pod in Running status

# T018: Verify listener pod
kubectl get pods -n arc-runners
# Expected: homelab-runners-*-listener pod in Running status

# T019-T020: Test workflow
# 1. Create .github/workflows/test-arc.yaml in any repository:
#
# name: Test ARC Runner
# on: workflow_dispatch
# jobs:
#   test:
#     runs-on: homelab
#     steps:
#       - run: echo "Running on homelab runner!"
#
# 2. Trigger the workflow manually
# 3. Verify runner pod is created and job completes
```

## Configuration

- **Controller namespace**: `arc-systems`
- **Runner namespace**: `arc-runners`
- **Runner label**: `homelab` (use `runs-on: homelab` in workflows)
- **Scaling**: 0-3 runners
- **Resources per runner**: 4 CPU, 8GB RAM
- **Container mode**: Kubernetes (no Docker daemon)

## Troubleshooting

See `specs/016-deploy-arc/quickstart.md` for detailed troubleshooting steps.

## User Story Progress

- ✅ US1: Setup and manifest creation (T001-T015)
- ⏳ US1: Deployment and verification (T016-T020) - Manual steps required
- ⏳ US2: Scaling behavior verification (T021-T026)
- ⏳ US3: GitOps flow verification (T027-T031)
