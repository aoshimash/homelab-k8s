# Data Model: Tailscale Kubernetes Operator

**Feature**: 003-tailscale-k8s-operator
**Date**: 2026-01-03

## Kubernetes Resources

### 1. Namespace

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: tailscale
```

**Purpose**: Dedicated namespace for Tailscale Operator and proxy resources

### 2. HelmRepository

```yaml
apiVersion: source.toolkit.fluxcd.io/v1
kind: HelmRepository
metadata:
  name: tailscale
  namespace: flux-system
spec:
  interval: 1h
  url: https://pkgs.tailscale.com/helmcharts
```

**Purpose**: Flux source for Tailscale Helm charts

### 3. OAuth Secret (SOPS-encrypted)

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: tailscale-operator-oauth
  namespace: tailscale
type: Opaque
stringData:
  client_id: <encrypted>
  client_secret: <encrypted>
```

**Fields**:
| Field | Type | Description | Source |
|-------|------|-------------|--------|
| client_id | string | OAuth client ID | Tailscale Admin Console |
| client_secret | string | OAuth client secret | Tailscale Admin Console |

**Encryption**: SOPS with Age key

### 4. HelmRelease

```yaml
apiVersion: helm.toolkit.fluxcd.io/v2
kind: HelmRelease
metadata:
  name: tailscale-operator
  namespace: tailscale
spec:
  chart:
    spec:
      chart: tailscale-operator
      sourceRef:
        kind: HelmRepository
        name: tailscale
        namespace: flux-system
      version: <pinned-version>
  values:
    oauth:
      clientId: # from secret
      clientSecret: # from secret
    operatorConfig:
      hostname: tailscale-operator
```

**Key Values**:
| Value Path | Type | Description |
|------------|------|-------------|
| oauth.clientId | string | OAuth client ID |
| oauth.clientSecret | string | OAuth client secret |
| operatorConfig.hostname | string | Operator device hostname in tailnet |

### 5. ProxyGroup (Optional)

```yaml
apiVersion: tailscale.com/v1alpha1
kind: ProxyGroup
metadata:
  name: ingress-proxies
  namespace: tailscale
spec:
  type: ingress
  replicas: 1
```

**Fields**:
| Field | Type | Description | Constraints |
|-------|------|-------------|-------------|
| type | enum | Proxy type | `ingress` or `egress` |
| replicas | integer | Number of proxy replicas | >= 1 |

**State Transitions**:
- `Pending` → `Ready` (all replicas healthy)
- `Ready` → `Degraded` (some replicas unhealthy)
- `Ready` → `Pending` (configuration change)

### 6. Ingress (Tailscale)

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: longhorn-ui
  namespace: longhorn-system
spec:
  ingressClassName: tailscale
  rules:
    - host: longhorn
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: longhorn-frontend
                port:
                  number: 80
```

**Fields**:
| Field | Type | Description |
|-------|------|-------------|
| ingressClassName | string | Must be `tailscale` |
| rules[].host | string | Hostname prefix (becomes `<host>.<tailnet>.ts.net`) |
| rules[].http.paths[].backend.service.name | string | Target service name |
| rules[].http.paths[].backend.service.port.number | integer | Target service port |

## Entity Relationships

```
┌─────────────────┐     ┌─────────────────┐
│  HelmRepository │────▶│   HelmRelease   │
│   (flux-system) │     │   (tailscale)   │
└─────────────────┘     └────────┬────────┘
                                 │
                                 │ deploys
                                 ▼
┌─────────────────┐     ┌─────────────────┐
│   OAuth Secret  │────▶│    Operator     │
│   (tailscale)   │     │     (Pod)       │
└─────────────────┘     └────────┬────────┘
                                 │
                                 │ manages
                                 ▼
┌─────────────────┐     ┌─────────────────┐
│   ProxyGroup    │◀────│    Ingress      │
│   (tailscale)   │     │ (any namespace) │
└────────┬────────┘     └─────────────────┘
         │
         │ creates
         ▼
┌─────────────────┐
│  StatefulSet    │
│  (proxy pods)   │
└─────────────────┘
```

## Tailnet Entities (External)

### OAuth Client

**Created in**: Tailscale Admin Console (manual step)

| Attribute | Value |
|-----------|-------|
| Scopes | `Devices Core` (write), `Auth Keys` (write), `Services` (write) |
| Tags | `tag:k8s-operator`, `tag:k8s` |

### Tailnet Devices

| Device | Tag | Created By |
|--------|-----|------------|
| tailscale-operator | tag:k8s-operator | Operator deployment |
| ingress-proxies-0 | tag:k8s | ProxyGroup StatefulSet |

## File Structure

```
k8s/infrastructure/tailscale/
├── kustomization.yaml
├── namespace.yaml
├── helmrepository.yaml
├── helmrelease.yaml
├── secret-oauth-credentials.sops.yaml
├── proxygroup.yaml
└── smoke/
    ├── kustomization.yaml
    └── ingress-longhorn.yaml
```
