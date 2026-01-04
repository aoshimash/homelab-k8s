# Research: Home Assistant on Kubernetes with Matter Integration

**Feature**: 011-home-assistant
**Date**: 2026-01-04

## Research Topics

### 1. Home Assistant Container Image Selection

**Decision**: Use official Home Assistant image `ghcr.io/home-assistant/home-assistant`

**Rationale**:
- Official image maintained by Home Assistant team
- Regular security updates and releases
- Built-in Matter integration support
- Well-documented and widely used in production

**Alternatives Considered**:
- LinuxServer.io image (`lscr.io/linuxserver/homeassistant`): Good for PUID/PGID control but adds unnecessary layer
- Custom build: Overkill for standard deployment

### 2. hostNetwork Mode for Matter/mDNS

**Decision**: Use `hostNetwork: true` with `dnsPolicy: ClusterFirstWithHostNet`

**Rationale**:
- Matter protocol requires mDNS (multicast DNS) for device discovery
- mDNS uses UDP port 5353 with multicast addresses
- IPv6 link-local addresses (fe80::) are required for Matter commissioning
- Standard Kubernetes pod networking does not support multicast properly
- `ClusterFirstWithHostNet` allows pod to use cluster DNS while on host network

**Alternatives Considered**:
- Macvlan CNI: Complex setup, not necessary for single-node cluster
- Host port mapping: Does not support multicast/mDNS properly
- Cilium L2 announcements: Does not solve mDNS multicast issue

### 3. Node Affinity Strategy

**Decision**: Use `nodeSelector` to pin pod to `homelab-node-01`

**Rationale**:
- Matter device pairing is tied to specific network interface/MAC address
- Moving pod to different node would require re-pairing all Matter devices
- Single-node cluster makes this straightforward
- Simplest solution with predictable behavior

**Alternatives Considered**:
- No affinity (let scheduler decide): Risk of re-pairing on reschedule
- Node affinity with preferred: Still allows unwanted rescheduling
- StatefulSet: Overkill for single replica

### 4. Storage Configuration

**Decision**: Single Longhorn PVC (5Gi) mounted at `/config`

**Rationale**:
- Home Assistant stores all data in `/config` directory
- Includes: configuration.yaml, automations, SQLite database, Matter credentials
- 5Gi sufficient for configuration, database, and Matter pairing data
- Longhorn provides data persistence and backup capability

**Alternatives Considered**:
- Multiple PVCs: Unnecessary complexity
- emptyDir: Data loss on pod restart
- hostPath: Violates immutable infrastructure principle

### 5. Tailscale Ingress Configuration

**Decision**: Use Tailscale Ingress with hostname `home-assistant`

**Rationale**:
- Consistent with existing pattern (audiobookshelf, longhorn)
- Provides secure access without exposing to public internet
- Uses existing `ingress-proxies` ProxyGroup
- Automatic TLS via Tailscale MagicDNS

**Configuration Pattern**:
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  annotations:
    tailscale.com/proxy-group: ingress-proxies
spec:
  ingressClassName: tailscale
  tls:
    - hosts:
        - home-assistant
```

### 6. ConfigMap for Automations

**Decision**: Mount automation.yaml via ConfigMap with subPath

**Rationale**:
- Allows Git-managed automation definitions
- Changes can be deployed via Flux without pod restart (with reload)
- Follows infrastructure-as-code principles
- Sample automation demonstrates pattern for users

**Implementation**:
- ConfigMap contains `automation.yaml` content
- Mounted to `/config/automations.yaml` using subPath
- Home Assistant's `automation: !include automations.yaml` loads it

**Alternatives Considered**:
- Direct configuration.yaml in ConfigMap: Too restrictive, breaks UI editing
- No ConfigMap: Loses Git tracking benefit
- Full /config as ConfigMap: Would conflict with PVC

### 7. Home Assistant Port Configuration

**Decision**: Container port 8123 (default), Service port 80

**Rationale**:
- 8123 is Home Assistant's default web interface port
- Service exposes as port 80 for cleaner URLs via Ingress
- Consistent with audiobookshelf pattern (container 80, service 80)

### 8. Security Context Considerations

**Decision**: Run as root (default) due to hostNetwork requirement

**Rationale**:
- hostNetwork mode requires elevated privileges
- Home Assistant official image expects root for initial setup
- Matter integration may require access to host network interfaces
- Security boundary is at Tailscale Ingress level

**Mitigations**:
- No privileged mode required
- Read-only root filesystem not possible (HA writes to /config)
- Network access restricted to Tailscale

## Summary of Key Decisions

| Topic | Decision | Key Reason |
|-------|----------|------------|
| Container Image | ghcr.io/home-assistant/home-assistant | Official, Matter support |
| Network Mode | hostNetwork: true | mDNS/Matter requirement |
| DNS Policy | ClusterFirstWithHostNet | Cluster DNS + host network |
| Node Affinity | nodeSelector: homelab-node-01 | Prevent Matter re-pairing |
| Storage | Longhorn PVC 5Gi | Persistence, backup |
| Ingress | Tailscale, hostname: home-assistant | Existing pattern |
| Automations | ConfigMap with subPath mount | Git-managed configs |
| Port | 8123 (container), 80 (service) | Standard pattern |
