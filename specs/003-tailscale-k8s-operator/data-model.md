# Data Model: Tailscale Kubernetes Operator Installation

**Feature**: 003-tailscale-k8s-operator
**Date**: 2026-01-02

## Kubernetes Resources

### 1. Namespace

| Field | Value | Description |
|-------|-------|-------------|
| name | `tailscale` | Dedicated namespace for Tailscale resources |
| labels.pod-security.kubernetes.io/enforce | `privileged` | Required for network configuration |

**State Transitions**: N/A (static resource)

### 2. HelmRepository

| Field | Value | Description |
|-------|-------|-------------|
| name | `tailscale` | Repository identifier |
| namespace | `flux-system` | Flux source namespace |
| spec.url | `https://pkgs.tailscale.com/helmcharts` | Official Tailscale Helm charts |
| spec.interval | `1h0m0s` | Chart metadata refresh interval |

**State Transitions**:
- `Unknown` → `Ready` (successful fetch)
- `Ready` → `Failed` (network/auth error)

### 3. HelmRelease

| Field | Value | Description |
|-------|-------|-------------|
| name | `tailscale-operator` | Release identifier |
| namespace | `tailscale` | Target namespace |
| spec.chart.spec.chart | `tailscale-operator` | Chart name |
| spec.chart.spec.version | `<pinned-version>` | Specific version for reproducibility |
| spec.chart.spec.sourceRef.name | `tailscale` | Reference to HelmRepository |
| spec.interval | `10m0s` | Reconciliation interval |

**Helm Values**:

| Value Path | Type | Description |
|------------|------|-------------|
| oauth.clientId | string | Tailscale OAuth client ID |
| oauth.clientSecret | string | Tailscale OAuth client secret |
| operatorConfig.hostname | string | Operator device hostname (default: `tailscale-operator`) |

**State Transitions**:
- `Unknown` → `Installing` → `Ready` (successful deployment)
- `Ready` → `Upgrading` → `Ready` (version update)
- `*` → `Failed` (chart/values error)

### 4. Secret (OAuth Credentials)

| Field | Value | Description |
|-------|-------|-------------|
| name | `tailscale-operator-oauth` | Secret identifier |
| namespace | `tailscale` | Same namespace as operator |
| type | `Opaque` | Generic secret type |

**Data Fields**:

| Key | Type | Description | Encrypted |
|-----|------|-------------|-----------|
| client_id | string | OAuth client ID from Tailscale admin console | Yes (SOPS) |
| client_secret | string | OAuth client secret from Tailscale admin console | Yes (SOPS) |

**Validation Rules**:
- `client_id` must be non-empty
- `client_secret` must be non-empty
- Both must match credentials created in Tailscale admin console

### 5. Gateway (Tailscale Gateway)

| Field | Value | Description |
|-------|-------|-------------|
| name | `tailscale-gateway` | Gateway identifier |
| namespace | `tailscale` | Tailscale namespace |
| spec.gatewayClassName | `tailscale` | Tailscale Gateway controller |

**Listeners**:

| Name | Protocol | Port | Description |
|------|----------|------|-------------|
| `https` | HTTPS | 443 | HTTPS listener with auto TLS |

**State Transitions**:
- `Unknown` → `Pending` (waiting for operator)
- `Pending` → `Accepted` → `Programmed` (gateway ready)
- `*` → `Failed` (operator error)

### 6. HTTPRoute (Longhorn UI)

| Field | Value | Description |
|-------|-------|-------------|
| name | `longhorn-ui` | HTTPRoute identifier |
| namespace | `longhorn-system` | Longhorn namespace |
| spec.parentRefs | `tailscale-gateway` in `tailscale` ns | Reference to Gateway |

**Rules**:

| Hostname | Path | Backend Service | Backend Port |
|----------|------|-----------------|--------------|
| `longhorn` | `/` (PathPrefix) | `longhorn-frontend` | `80` |

**State Transitions**:
- `Unknown` → `Pending` (waiting for gateway)
- `Pending` → `Accepted` (route accepted by gateway)
- `Accepted` → `Programmed` (proxy deployed, device registered)
- `*` → `Failed` (operator error)

## Entity Relationships

```
┌─────────────────────────────────────────────────────────────┐
│                      flux-system namespace                   │
│  ┌─────────────────┐                                        │
│  │  HelmRepository │ ─────────────────────────────┐         │
│  │   (tailscale)   │                              │         │
│  └─────────────────┘                              │         │
└───────────────────────────────────────────────────│─────────┘
                                                    │
                                                    │ sourceRef
                                                    ▼
┌─────────────────────────────────────────────────────────────┐
│                      tailscale namespace                     │
│  ┌─────────────────┐      ┌─────────────────────────────┐   │
│  │    Namespace    │      │        HelmRelease          │   │
│  │   (tailscale)   │◄─────│   (tailscale-operator)      │   │
│  └─────────────────┘      └──────────────┬──────────────┘   │
│                                          │                   │
│                                          │ valuesFrom        │
│                                          ▼                   │
│                           ┌─────────────────────────────┐   │
│                           │          Secret             │   │
│                           │ (tailscale-operator-oauth)  │   │
│                           └─────────────────────────────┘   │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│                      tailscale namespace                     │
│                                                              │
│  ┌─────────────────────────────────────┐                    │
│  │           Gateway                   │                    │
│  │    (tailscale-gateway)              │                    │
│  │    gatewayClassName: tailscale      │                    │
│  └──────────────┬──────────────────────┘                    │
│                 │                                            │
│                 │ parentRef                                  │
│                 ▼                                            │
│  ┌─────────────────────────────────────┐                    │
│  │  Tailscale Proxy (auto-created)     │                    │
│  │  - StatefulSet in tailscale ns      │                    │
│  │  - Device in Tailscale admin        │                    │
│  └─────────────────────────────────────┘                    │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│                   longhorn-system namespace                  │
│  ┌─────────────────┐      ┌─────────────────────────────┐   │
│  │    HTTPRoute    │─────►│        Service              │   │
│  │  (longhorn-ui)  │      │   (longhorn-frontend)       │   │
│  └─────────────────┘      └─────────────────────────────┘   │
│         │                                                    │
│         │ parentRef: tailscale-gateway (tailscale ns)        │
│         │ (watched by Tailscale Operator)                    │
│         │                                                    │
└─────────────────────────────────────────────────────────────┘
```

## External Entities (Tailscale Control Plane)

### OAuth Client (Tailscale Admin Console)

| Attribute | Value | Description |
|-----------|-------|-------------|
| Scopes | Devices Core, Auth Keys, Services (write) | Required permissions |
| Tags | `tag:k8s-operator` | Device tag for ACL management |

### Tailnet Devices (Created by Operator)

| Device | Description |
|--------|-------------|
| `tailscale-operator` | Main operator device |
| `longhorn-<suffix>` | Gateway proxy for Longhorn UI HTTPRoute |

## File Mapping

| Entity | File Path |
|--------|-----------|
| Namespace | `k8s/infrastructure/tailscale/namespace.yaml` |
| HelmRepository | `k8s/infrastructure/tailscale/helmrepository.yaml` |
| HelmRelease | `k8s/infrastructure/tailscale/helmrelease.yaml` |
| Secret | `k8s/infrastructure/tailscale/secret-oauth.sops.yaml` |
| Gateway | `k8s/infrastructure/tailscale/gateway.yaml` |
| HTTPRoute | `k8s/infrastructure/tailscale/httproute-longhorn.yaml` |
