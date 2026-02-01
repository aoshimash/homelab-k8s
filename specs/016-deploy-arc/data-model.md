# Data Model: Deploy Actions Runner Controller (ARC)

**Date**: 2026-02-01  
**Feature**: 016-deploy-arc

## Kubernetes Resources

### 1. Namespaces

#### arc-systems (Controller Namespace)
```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: arc-systems
  labels:
    app.kubernetes.io/part-of: actions-runner-controller
```

#### arc-runners (Runner Namespace)
```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: arc-runners
  labels:
    app.kubernetes.io/part-of: actions-runner-controller
```

### 2. HelmRepository (OCI)

```yaml
apiVersion: source.toolkit.fluxcd.io/v1
kind: HelmRepository
metadata:
  name: actions-runner-controller
  namespace: flux-system
spec:
  type: oci
  interval: 1h
  url: oci://ghcr.io/actions/actions-runner-controller-charts
```

### 3. HelmRelease - Controller

```yaml
apiVersion: helm.toolkit.fluxcd.io/v2
kind: HelmRelease
metadata:
  name: arc-controller
  namespace: arc-systems
spec:
  interval: 10m
  chart:
    spec:
      chart: gha-runner-scale-set-controller
      sourceRef:
        kind: HelmRepository
        name: actions-runner-controller
        namespace: flux-system
  values:
    # Controller-specific configuration
```

### 4. HelmRelease - Runner Scale Set

```yaml
apiVersion: helm.toolkit.fluxcd.io/v2
kind: HelmRelease
metadata:
  name: homelab-runners
  namespace: arc-runners
spec:
  interval: 10m
  chart:
    spec:
      chart: gha-runner-scale-set
      sourceRef:
        kind: HelmRepository
        name: actions-runner-controller
        namespace: flux-system
  values:
    githubConfigUrl: "https://github.com/<ORGANIZATION>"
    githubConfigSecret: arc-github-app
    minRunners: 0
    maxRunners: 3
    runnerScaleSetName: homelab
    containerMode:
      type: kubernetes
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

### 5. Secret - GitHub App Credentials

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: arc-github-app
  namespace: arc-runners
type: Opaque
stringData:
  github_app_id: "<APP_ID>"
  github_app_installation_id: "<INSTALLATION_ID>"
  github_app_private_key: |
    -----BEGIN RSA PRIVATE KEY-----
    <PRIVATE_KEY_CONTENT>
    -----END RSA PRIVATE KEY-----
```

## Entity Relationships

```
┌─────────────────────────────────────────────────────────────────┐
│                        flux-system                               │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │              HelmRepository (OCI)                        │    │
│  │   actions-runner-controller                              │    │
│  │   → ghcr.io/actions/actions-runner-controller-charts     │    │
│  └─────────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────────┘
                              │
                              │ sourceRef
                    ┌─────────┴─────────┐
                    ▼                   ▼
┌───────────────────────────┐  ┌───────────────────────────┐
│       arc-systems         │  │       arc-runners         │
│  ┌─────────────────────┐  │  │  ┌─────────────────────┐  │
│  │    HelmRelease      │  │  │  │    HelmRelease      │  │
│  │    arc-controller   │  │  │  │    homelab-runners  │  │
│  └─────────────────────┘  │  │  └──────────┬──────────┘  │
│           │               │  │             │             │
│           ▼               │  │             ▼             │
│  ┌─────────────────────┐  │  │  ┌─────────────────────┐  │
│  │ Controller Pod      │  │  │  │ Secret              │  │
│  │ (manages runners)   │──┼──┼──│ arc-github-app      │  │
│  └─────────────────────┘  │  │  └─────────────────────┘  │
│                           │  │             │             │
│                           │  │             ▼             │
│                           │  │  ┌─────────────────────┐  │
│                           │  │  │ Listener Pod        │  │
│                           │  │  │ (receives jobs)     │  │
│                           │  │  └──────────┬──────────┘  │
│                           │  │             │             │
│                           │  │             ▼             │
│                           │  │  ┌─────────────────────┐  │
│                           │  │  │ Runner Pods (0-3)   │  │
│                           │  │  │ (execute jobs)      │  │
│                           │  │  └─────────────────────┘  │
└───────────────────────────┘  └───────────────────────────┘
```

## Custom Resource Definitions (Managed by Controller)

The ARC controller installs the following CRDs:

| CRD | Purpose |
|-----|---------|
| `AutoscalingRunnerSet` | Defines a set of runners with scaling config |
| `AutoscalingListener` | Manages the listener that receives jobs from GitHub |
| `EphemeralRunnerSet` | Manages ephemeral runner pods |
| `EphemeralRunner` | Individual runner instance |

These are managed internally by the Helm chart and do not require direct manipulation.

## FluxCD Kustomization Dependencies

```yaml
# k8s/flux/infrastructure-kustomization.yaml
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: infrastructure
spec:
  # ... existing config ...
  # ARC controller included here

---
# k8s/flux/configs-kustomization.yaml  
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: configs
spec:
  dependsOn:
    - name: infrastructure  # Ensures controller is ready first
  # ... existing config ...
  # ARC runner scale set included here
```

## Validation Rules

| Resource | Validation |
|----------|------------|
| Secret | Must contain all three fields: `github_app_id`, `github_app_installation_id`, `github_app_private_key` |
| HelmRelease (runner) | `githubConfigUrl` must be valid GitHub org URL |
| HelmRelease (runner) | `minRunners` ≤ `maxRunners` |
| HelmRelease (runner) | `runnerScaleSetName` must match workflow `runs-on` value |

## State Transitions

### Runner Pod Lifecycle

```
                    Job Queued
                        │
                        ▼
┌──────────┐      ┌──────────┐      ┌──────────┐
│  None    │ ──▶  │ Pending  │ ──▶  │ Running  │
│ (scaled  │      │ (pod     │      │ (job     │
│  to 0)   │      │ creating)│      │ executing│
└──────────┘      └──────────┘      └──────────┘
     ▲                                    │
     │                                    ▼
     │                             ┌──────────┐
     └──────────────────────────── │Terminated│
              (scale down)         │ (cleanup)│
                                   └──────────┘
```
