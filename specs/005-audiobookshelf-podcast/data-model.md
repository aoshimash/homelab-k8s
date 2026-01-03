# Data Model: Audiobookshelf Podcast Server

**Feature**: 005-audiobookshelf-podcast
**Date**: 2026-01-03

## Overview

This document describes the Kubernetes resources and their relationships for the Audiobookshelf deployment. Audiobookshelf manages its own internal data model (podcasts, episodes, users) in SQLite; this document focuses on the Kubernetes resource model.

## Kubernetes Resources

### Namespace

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: audiobookshelf
```

**Purpose**: Isolate Audiobookshelf resources from other workloads.

### PersistentVolumeClaim: Config

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: audiobookshelf-config
  namespace: audiobookshelf
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: longhorn
  resources:
    requests:
      storage: 1Gi
```

**Purpose**: Store Audiobookshelf configuration, SQLite database, and metadata cache.

| Field | Value | Description |
|-------|-------|-------------|
| name | audiobookshelf-config | Identifies the config volume |
| storage | 1Gi | Sufficient for database and metadata |
| storageClassName | longhorn | Use Longhorn for persistent storage |
| accessModes | ReadWriteOnce | Single pod access |

### PersistentVolumeClaim: Podcasts

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: audiobookshelf-podcasts
  namespace: audiobookshelf
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: longhorn
  resources:
    requests:
      storage: 50Gi
```

**Purpose**: Store downloaded podcast audio files.

| Field | Value | Description |
|-------|-------|-------------|
| name | audiobookshelf-podcasts | Identifies the podcast storage volume |
| storage | 50Gi | Per clarification, sufficient for podcast library |
| storageClassName | longhorn | Use Longhorn for persistent storage |
| accessModes | ReadWriteOnce | Single pod access |

### Deployment

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: audiobookshelf
  namespace: audiobookshelf
spec:
  replicas: 1
  selector:
    matchLabels:
      app: audiobookshelf
  template:
    metadata:
      labels:
        app: audiobookshelf
    spec:
      containers:
        - name: audiobookshelf
          image: ghcr.io/advplyr/audiobookshelf:latest
          ports:
            - containerPort: 80
          env:
            - name: AUDIOBOOKSHELF_UID
              value: "99"
            - name: AUDIOBOOKSHELF_GID
              value: "99"
          volumeMounts:
            - name: config
              mountPath: /config
            - name: podcasts
              mountPath: /podcasts
          resources:
            requests:
              cpu: 100m
              memory: 256Mi
            limits:
              cpu: 1000m
              memory: 1Gi
          livenessProbe:
            httpGet:
              path: /healthcheck
              port: 80
            initialDelaySeconds: 30
            periodSeconds: 30
          readinessProbe:
            httpGet:
              path: /healthcheck
              port: 80
            initialDelaySeconds: 10
            periodSeconds: 10
      volumes:
        - name: config
          persistentVolumeClaim:
            claimName: audiobookshelf-config
        - name: podcasts
          persistentVolumeClaim:
            claimName: audiobookshelf-podcasts
```

**Purpose**: Run the Audiobookshelf application.

| Field | Value | Description |
|-------|-------|-------------|
| replicas | 1 | Single instance (SQLite doesn't support concurrent writes) |
| image | ghcr.io/advplyr/audiobookshelf:latest | Official container image |
| containerPort | 80 | HTTP web interface |
| volumeMounts | /config, /podcasts | Persistent storage locations |

### Service

```yaml
apiVersion: v1
kind: Service
metadata:
  name: audiobookshelf
  namespace: audiobookshelf
spec:
  type: ClusterIP
  selector:
    app: audiobookshelf
  ports:
    - port: 80
      targetPort: 80
      protocol: TCP
```

**Purpose**: Expose the Audiobookshelf pod within the cluster.

| Field | Value | Description |
|-------|-------|-------------|
| type | ClusterIP | Internal-only service |
| port | 80 | Service port |
| selector | app: audiobookshelf | Routes to Audiobookshelf pods |

### Ingress

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: audiobookshelf
  namespace: audiobookshelf
  annotations:
    tailscale.com/proxy-group: ingress-proxies
spec:
  ingressClassName: tailscale
  defaultBackend:
    service:
      name: audiobookshelf
      port:
        number: 80
  tls:
    - hosts:
        - audiobookshelf
```

**Purpose**: Expose Audiobookshelf to the tailnet via Tailscale Ingress.

| Field | Value | Description |
|-------|-------|-------------|
| ingressClassName | tailscale | Use Tailscale Ingress controller |
| proxy-group | ingress-proxies | Use existing ProxyGroup |
| tls.hosts | audiobookshelf | Hostname on tailnet |

## Resource Relationships

```
┌─────────────────────────────────────────────────────────────┐
│                    Namespace: audiobookshelf                 │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  ┌─────────────┐        ┌────────────────────────────────┐ │
│  │   Ingress   │───────▶│            Service             │ │
│  │ (tailscale) │        │        (ClusterIP:80)          │ │
│  └─────────────┘        └────────────┬───────────────────┘ │
│        │                             │                      │
│        │                             ▼                      │
│        │                ┌────────────────────────────────┐ │
│        │                │          Deployment            │ │
│        │                │    ┌────────────────────┐     │ │
│        │                │    │   Pod: audiobookshelf │   │ │
│        │                │    │   ┌──────────────┐  │     │ │
│        │                │    │   │  Container   │  │     │ │
│        │                │    │   │  Port: 80    │  │     │ │
│        │                │    │   └──────┬───────┘  │     │ │
│        │                │    └──────────┼──────────┘     │ │
│        │                └───────────────┼────────────────┘ │
│        │                                │                   │
│        │                    ┌───────────┴───────────┐      │
│        │                    │                       │      │
│        ▼                    ▼                       ▼      │
│  ┌───────────┐    ┌─────────────────┐    ┌──────────────┐ │
│  │ProxyGroup │    │      PVC        │    │     PVC      │ │
│  │ (tailscale│    │ config (1Gi)    │    │podcasts(50Gi)│ │
│  │ namespace)│    └────────┬────────┘    └──────┬───────┘ │
│  └───────────┘             │                    │         │
│                            ▼                    ▼         │
│                   ┌─────────────────────────────────────┐ │
│                   │        Longhorn StorageClass        │ │
│                   └─────────────────────────────────────┘ │
│                                                             │
└─────────────────────────────────────────────────────────────┘

External Access Flow:
┌──────────────┐     ┌───────────┐     ┌─────────┐     ┌─────┐
│ Tailnet      │────▶│ ProxyGroup│────▶│ Ingress │────▶│ Svc │
│ Device       │     │ (HTTPS)   │     │         │     │     │
└──────────────┘     └───────────┘     └─────────┘     └─────┘
```

## Flux Kustomization Structure

```
k8s/
├── apps/
│   ├── kustomization.yaml          # Apps root kustomization
│   └── audiobookshelf/
│       ├── kustomization.yaml      # Audiobookshelf kustomization
│       ├── namespace.yaml
│       ├── pvc.yaml                # Both PVCs in one file
│       ├── deployment.yaml
│       ├── service.yaml
│       └── ingress.yaml
└── flux/
    └── apps-kustomization.yaml     # Flux Kustomization for apps
```

## State Transitions

Audiobookshelf manages its own application state internally. From a Kubernetes perspective:

| State | Condition | Description |
|-------|-----------|-------------|
| Pending | PVCs not bound | Waiting for Longhorn to provision storage |
| Starting | Pod initializing | Container starting, probes not yet passing |
| Ready | Readiness probe passing | Application accepting traffic |
| Running | All probes passing | Normal operation |
| Degraded | Liveness probe failing | Application unresponsive, will restart |
