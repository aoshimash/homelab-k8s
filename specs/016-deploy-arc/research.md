# Research: Deploy Actions Runner Controller (ARC)

**Date**: 2026-02-01  
**Feature**: 016-deploy-arc

## 1. ARC Architecture Decision

### Decision: Use Runner Scale Sets (Modern Mode)

**Rationale**: 
- Runner Scale Sets is the officially supported mode by GitHub
- Legacy RunnerDeployment/HRA mode is community-maintained only
- Scale Sets provide better integration with GitHub Actions queue
- Automatic scaling based on actual job queue, not webhook events

**Alternatives Considered**:
- Legacy mode (RunnerDeployment + HorizontalRunnerAutoscaler): Rejected - community-only support, webhook-based scaling is less accurate
- Summerwind/actions-runner-controller fork: Rejected - deprecated in favor of official GitHub-supported ARC

## 2. Helm Chart Source

### Decision: Use OCI Registry (`ghcr.io/actions/actions-runner-controller-charts`)

**Rationale**:
- Official distribution channel from GitHub
- OCI registry is the modern Helm standard
- FluxCD HelmRepository supports OCI registries natively

**Alternatives Considered**:
- Traditional Helm repository: Not available for modern ARC
- Raw manifests: Rejected - more maintenance burden, no templating

## 3. Namespace Strategy

### Decision: Two Separate Namespaces

| Namespace | Purpose | Contents |
|-----------|---------|----------|
| `arc-systems` | Controller components | gha-runner-scale-set-controller pods |
| `arc-runners` | Runner pods | Ephemeral runner pods, listener pods |

**Rationale**:
- Security best practice from official documentation
- Controller has elevated permissions (CRD management)
- Runners should be isolated from controller
- Easier RBAC management and resource quotas

**Alternatives Considered**:
- Single namespace: Rejected - violates security best practices
- Multiple runner namespaces: Not needed for single scale set

## 4. GitHub App Configuration

### Decision: Organization-Level GitHub App

**Required Permissions**:
| Scope | Permission | Level |
|-------|------------|-------|
| Organization | Self-hosted runners | Read and write |
| Repository | Metadata | Read-only |

**Rationale**:
- Organization-level allows all repos to use runners
- GitHub App provides automatic token refresh (no manual rotation)
- More secure than PAT (scoped permissions, audit logs)

**Credentials Required**:
- `github_app_id`: Numeric App ID from GitHub App settings
- `github_app_installation_id`: Installation ID for the organization
- `github_app_private_key`: PEM-formatted private key

## 5. FluxCD Integration Pattern

### Decision: Standard HelmRelease Pattern

**Components**:
1. **HelmRepository** (type: oci) - Points to `ghcr.io/actions/actions-runner-controller-charts`
2. **HelmRelease (controller)** - Deploys gha-runner-scale-set-controller
3. **HelmRelease (runner-set)** - Deploys gha-runner-scale-set with homelab config

**Rationale**:
- Consistent with existing infrastructure patterns (tailscale, cloudnative-pg)
- Native FluxCD reconciliation and drift detection
- SOPS decryption built into Flux Kustomization

**Alternatives Considered**:
- Kustomize with raw manifests: Rejected - loses Helm templating benefits
- ArgoCD: Not applicable - cluster uses FluxCD

## 6. Runner Configuration

### Decision: Kubernetes Mode (Container-based)

**Configuration**:
```yaml
containerMode:
  type: "kubernetes"
  kubernetesModeWorkVolumeClaim:
    accessModes: ["ReadWriteOnce"]
    storageClassName: "longhorn"  # Use existing storage class
    resources:
      requests:
        storage: 10Gi
```

**Rationale**:
- No Docker daemon required (no privileged containers)
- Better security isolation
- Each workflow step runs in separate container
- Works with existing Longhorn storage

**Alternatives Considered**:
- Docker-in-Docker (DinD): Rejected - requires privileged containers, security risk
- Docker sidecar: Rejected - same security concerns

## 7. Scaling Configuration

### Decision: 0-3 Runners with Default Scale Set Behavior

**Configuration**:
```yaml
minRunners: 0
maxRunners: 3
```

**Rationale**:
- minRunners: 0 saves resources when idle (homelab constraint)
- maxRunners: 3 balances parallelism with resource limits
- Total max resources: 12 CPU, 24GB RAM (within homelab capacity)

## 8. Resource Limits

### Decision: 4 CPU / 8GB RAM per Runner

**Configuration**:
```yaml
template:
  spec:
    containers:
    - name: runner
      resources:
        limits:
          cpu: "4"
          memory: "8Gi"
        requests:
          cpu: "2"
          memory: "4Gi"
```

**Rationale**:
- Supports heavy builds (compilation, Docker builds via kaniko)
- Requests lower than limits for bin packing efficiency
- Aligns with clarification session decision

## 9. Secret Management

### Decision: SOPS-Encrypted Secrets in Git

**File**: `secret-github-app.sops.yaml`

**Structure**:
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: arc-github-app
  namespace: arc-runners
type: Opaque
stringData:
  github_app_id: "123456"
  github_app_installation_id: "654321"
  github_app_private_key: |
    -----BEGIN RSA PRIVATE KEY-----
    ...
    -----END RSA PRIVATE KEY-----
```

**Rationale**:
- Consistent with existing homelab patterns (tailscale, grafana-cloud)
- Flux Kustomization decrypts at reconcile time
- No plaintext secrets in repository

## 10. Dependencies and Ordering

### FluxCD Dependency Chain

```
infrastructure/actions-runner-controller (controller)
    ↓ depends on
configs/arc-runners (runner scale set)
```

**Implementation**:
- Use `dependsOn` in Flux Kustomization
- Controller must be healthy before runner set deploys

## Summary of Resolved Items

| Item | Resolution |
|------|------------|
| ARC Mode | Runner Scale Sets (modern, GitHub-supported) |
| Chart Source | OCI registry (ghcr.io) |
| Namespaces | arc-systems (controller), arc-runners (runners) |
| Authentication | GitHub App (organization-level) |
| FluxCD Pattern | HelmRepository + HelmRelease |
| Container Mode | Kubernetes mode (no Docker daemon) |
| Scaling | 0-3 runners |
| Resources | 4 CPU / 8GB RAM limits per runner |
| Secrets | SOPS-encrypted in Git |
