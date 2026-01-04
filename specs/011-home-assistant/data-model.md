# Data Model: Home Assistant Kubernetes Deployment

**Feature**: 011-home-assistant
**Date**: 2026-01-04

## Kubernetes Resources

### 1. Namespace

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: home-assistant
```

**Purpose**: Isolate Home Assistant resources from other applications.

### 2. PersistentVolumeClaim

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: home-assistant-config
  namespace: home-assistant
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: longhorn
  resources:
    requests:
      storage: 5Gi
```

**Fields**:
| Field | Value | Rationale |
|-------|-------|-----------|
| accessModes | ReadWriteOnce | Single pod access, standard for stateful apps |
| storageClassName | longhorn | Existing storage solution with backup support |
| storage | 5Gi | Sufficient for config, DB, Matter credentials |

### 3. ConfigMap (Automations)

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: home-assistant-automations
  namespace: home-assistant
data:
  automations.yaml: |
    # Sample automation - turn on notification at startup
    - id: startup_notification
      alias: "Startup Notification"
      trigger:
        - platform: homeassistant
          event: start
      action:
        - service: persistent_notification.create
          data:
            title: "Home Assistant Started"
            message: "Home Assistant has started successfully."
```

**Purpose**: Git-managed automation definitions that can be updated via Flux.

### 4. Deployment

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: home-assistant
  namespace: home-assistant
spec:
  replicas: 1
  selector:
    matchLabels:
      app: home-assistant
  template:
    metadata:
      labels:
        app: home-assistant
    spec:
      hostNetwork: true
      dnsPolicy: ClusterFirstWithHostNet
      nodeSelector:
        kubernetes.io/hostname: homelab-node-01
      containers:
        - name: home-assistant
          image: ghcr.io/home-assistant/home-assistant:2024.12
          ports:
            - containerPort: 8123
              hostPort: 8123
          volumeMounts:
            - name: config
              mountPath: /config
            - name: automations
              mountPath: /config/automations.yaml
              subPath: automations.yaml
          resources:
            requests:
              cpu: 100m
              memory: 512Mi
            limits:
              cpu: 2000m
              memory: 2Gi
          livenessProbe:
            httpGet:
              path: /
              port: 8123
            initialDelaySeconds: 60
            periodSeconds: 30
          readinessProbe:
            httpGet:
              path: /
              port: 8123
            initialDelaySeconds: 30
            periodSeconds: 10
      volumes:
        - name: config
          persistentVolumeClaim:
            claimName: home-assistant-config
        - name: automations
          configMap:
            name: home-assistant-automations
```

**Key Configuration**:
| Field | Value | Rationale |
|-------|-------|-----------|
| hostNetwork | true | Required for mDNS/Matter |
| dnsPolicy | ClusterFirstWithHostNet | Cluster DNS with host network |
| nodeSelector | homelab-node-01 | Prevent Matter re-pairing |
| replicas | 1 | Single instance required |
| resources.requests.memory | 512Mi | Base memory for HA |
| resources.limits.memory | 2Gi | Allow for integrations |

### 5. Service

```yaml
apiVersion: v1
kind: Service
metadata:
  name: home-assistant
  namespace: home-assistant
spec:
  type: ClusterIP
  selector:
    app: home-assistant
  ports:
    - port: 80
      targetPort: 8123
      protocol: TCP
```

**Note**: With hostNetwork, the service is primarily for Ingress routing. The pod is also accessible directly on the node's IP:8123.

### 6. Ingress

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: home-assistant
  namespace: home-assistant
  annotations:
    tailscale.com/proxy-group: ingress-proxies
spec:
  ingressClassName: tailscale
  defaultBackend:
    service:
      name: home-assistant
      port:
        number: 80
  tls:
    - hosts:
        - home-assistant
```

**Access URL**: `https://home-assistant.<tailnet-name>.ts.net`

## Entity Relationships

```
┌─────────────────────────────────────────────────────────────────┐
│                     Namespace: home-assistant                    │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ┌──────────────┐      ┌──────────────────────────────────────┐ │
│  │   Ingress    │──────│              Service                 │ │
│  │              │      │  (ClusterIP, port 80 → 8123)         │ │
│  └──────────────┘      └──────────────┬───────────────────────┘ │
│                                       │                          │
│                                       ▼                          │
│  ┌────────────────────────────────────────────────────────────┐ │
│  │                      Deployment                             │ │
│  │  ┌──────────────────────────────────────────────────────┐  │ │
│  │  │                    Pod                                │  │ │
│  │  │  hostNetwork: true                                    │  │ │
│  │  │  nodeSelector: homelab-node-01                        │  │ │
│  │  │                                                       │  │ │
│  │  │  ┌─────────────────┐  ┌─────────────────────────────┐│  │ │
│  │  │  │  Container      │  │        Volume Mounts        ││  │ │
│  │  │  │  home-assistant │  │  /config ← PVC              ││  │ │
│  │  │  │  :8123          │  │  /config/automations.yaml   ││  │ │
│  │  │  └─────────────────┘  │     ← ConfigMap (subPath)   ││  │ │
│  │  │                       └─────────────────────────────┘│  │ │
│  │  └──────────────────────────────────────────────────────┘  │ │
│  └────────────────────────────────────────────────────────────┘ │
│                                                                  │
│  ┌──────────────────┐         ┌─────────────────────────────┐   │
│  │       PVC        │         │         ConfigMap           │   │
│  │  5Gi Longhorn    │         │  home-assistant-automations │   │
│  └──────────────────┘         └─────────────────────────────┘   │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
                                │
                                │ mDNS/IPv6 (hostNetwork)
                                ▼
                    ┌───────────────────────┐
                    │   SwitchBot Hub 2     │
                    │   (Matter Device)     │
                    └───────────────────────┘
```

## State Transitions

### Home Assistant Pod Lifecycle

```
┌─────────┐    ┌─────────────┐    ┌─────────┐    ┌──────────┐
│ Pending │───▶│ Initializing│───▶│ Running │───▶│Terminating│
└─────────┘    └─────────────┘    └─────────┘    └──────────┘
     │              │                  │               │
     │              │                  │               │
     ▼              ▼                  ▼               ▼
  PVC Bound    Config Loaded    Matter Ready    Graceful Stop
               Probes Pass      mDNS Active
```

### Matter Device States

```
┌────────────┐    ┌───────────┐    ┌────────────┐    ┌───────────┐
│ Discovered │───▶│ Pairing   │───▶│ Configured │───▶│ Operational│
└────────────┘    └───────────┘    └────────────┘    └───────────┘
     │                 │                  │                │
     │                 │                  │                │
     ▼                 ▼                  ▼                ▼
  mDNS Found      Code Entry        Credentials       Control
                                    Stored in PVC     Available
```

## Validation Rules

| Resource | Rule | Enforcement |
|----------|------|-------------|
| PVC | Must be bound before pod starts | Kubernetes scheduler |
| Deployment | replicas must be 1 | Manifest definition |
| Pod | Must run on homelab-node-01 | nodeSelector |
| ConfigMap | automations.yaml must be valid YAML | Home Assistant startup |
| Ingress | hostname must match TLS host | Tailscale operator |
