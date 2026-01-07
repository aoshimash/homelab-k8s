# Data Model: CloudNativePG PostgreSQL Cluster

**Feature**: 012-cloudnative-pg  
**Date**: 2026-01-07

## Kubernetes Resources

### 1. CloudNativePG Operator (k8s/infrastructure/cloudnative-pg/)

#### Namespace
```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: cnpg-system
```

| Field | Value | Notes |
|-------|-------|-------|
| name | cnpg-system | Standard CloudNativePG operator namespace |

#### HelmRepository
```yaml
apiVersion: source.toolkit.fluxcd.io/v1
kind: HelmRepository
metadata:
  name: cloudnative-pg
  namespace: flux-system
```

| Field | Value | Notes |
|-------|-------|-------|
| name | cloudnative-pg | Repository identifier |
| namespace | flux-system | Standard for Flux HelmRepositories |
| url | https://cloudnative-pg.github.io/charts | Official chart repository |

#### HelmRelease
```yaml
apiVersion: helm.toolkit.fluxcd.io/v2
kind: HelmRelease
metadata:
  name: cloudnative-pg
  namespace: cnpg-system
```

| Field | Value | Notes |
|-------|-------|-------|
| chart | cloudnative-pg | Chart name |
| version | (latest stable) | Pinned, Renovate updates |
| sourceRef | cloudnative-pg HelmRepository | In flux-system |

### 2. PostgreSQL Cluster (k8s/configs/postgres/)

#### Namespace
```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: postgres
```

| Field | Value | Notes |
|-------|-------|-------|
| name | postgres | Dedicated namespace for PostgreSQL cluster |

#### Cluster CRD
```yaml
apiVersion: postgresql.cnpg.io/v1
kind: Cluster
metadata:
  name: postgres-cluster
  namespace: postgres
```

| Field | Value | Notes |
|-------|-------|-------|
| instances | 1 | Single instance for homelab |
| imageName | ghcr.io/cloudnative-pg/postgresql:16 | PostgreSQL 16 |
| storage.size | 10Gi | From clarification |
| storage.storageClass | longhorn | Existing storage |
| backup.retentionPolicy | 7d | From clarification |
| backup.barmanObjectStore | R2 config | S3-compatible backup |

#### ScheduledBackup CRD
```yaml
apiVersion: postgresql.cnpg.io/v1
kind: ScheduledBackup
metadata:
  name: postgres-cluster-daily
  namespace: postgres
```

| Field | Value | Notes |
|-------|-------|-------|
| schedule | 0 0 0 * * * | Daily at midnight UTC |
| backupOwnerReference | cluster | Links to Cluster for retention |
| cluster.name | postgres-cluster | Target cluster |

#### Secret (R2 Credentials)
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: postgres-r2-credentials
  namespace: postgres
type: Opaque
```

| Field | Key | Notes |
|-------|-----|-------|
| stringData | ACCESS_KEY_ID | R2 access key |
| stringData | ACCESS_SECRET_KEY | R2 secret key |

## Auto-Generated Resources

CloudNativePG automatically creates these resources:

### Services

| Service Name | Type | Purpose |
|--------------|------|---------|
| postgres-cluster-rw | ClusterIP | Read-write endpoint (primary) |
| postgres-cluster-r | ClusterIP | Read-only endpoint |
| postgres-cluster-ro | ClusterIP | Read-only endpoint (alias) |

### Secrets (Auto-generated)

| Secret Name | Contents | Notes |
|-------------|----------|-------|
| postgres-cluster-superuser | Superuser credentials | username, password, connection string |
| postgres-cluster-app | App user credentials | Default app user |

### PersistentVolumeClaim

| PVC Name | Size | StorageClass |
|----------|------|--------------|
| postgres-cluster-1 | 10Gi | longhorn |

## Entity Relationships

```
┌─────────────────────────────────────────────────────────────────┐
│                        flux-system namespace                     │
│  ┌──────────────────┐                                           │
│  │  HelmRepository  │ cloudnative-pg                            │
│  │  (charts source) │                                           │
│  └────────┬─────────┘                                           │
└───────────│─────────────────────────────────────────────────────┘
            │ references
            ▼
┌─────────────────────────────────────────────────────────────────┐
│                       cnpg-system namespace                      │
│  ┌──────────────────┐      ┌─────────────────────────────────┐  │
│  │   HelmRelease    │─────▶│  CloudNativePG Operator         │  │
│  │ cloudnative-pg   │      │  (Controller Manager Pod)       │  │
│  └──────────────────┘      └──────────────┬──────────────────┘  │
└───────────────────────────────────────────│─────────────────────┘
                                            │ manages
                                            ▼
┌─────────────────────────────────────────────────────────────────┐
│                        postgres namespace                        │
│  ┌──────────────────┐      ┌─────────────────────────────────┐  │
│  │     Cluster      │─────▶│  PostgreSQL Pod                 │  │
│  │ postgres-cluster │      │  (postgres-cluster-1)           │  │
│  └────────┬─────────┘      └──────────────┬──────────────────┘  │
│           │                               │                      │
│           │ configures                    │ uses                 │
│           ▼                               ▼                      │
│  ┌──────────────────┐      ┌─────────────────────────────────┐  │
│  │ ScheduledBackup  │      │  PVC (postgres-cluster-1)       │  │
│  │ postgres-cluster │      │  10Gi on Longhorn               │  │
│  │     -daily       │      └─────────────────────────────────┘  │
│  └────────┬─────────┘                                           │
│           │                ┌─────────────────────────────────┐  │
│           │ uses           │  Services                       │  │
│           ▼                │  - postgres-cluster-rw          │  │
│  ┌──────────────────┐      │  - postgres-cluster-r           │  │
│  │     Secret       │      │  - postgres-cluster-ro          │  │
│  │ postgres-r2-     │      └─────────────────────────────────┘  │
│  │  credentials     │                                           │
│  └──────────────────┘      ┌─────────────────────────────────┐  │
│                            │  Secrets (auto-generated)       │  │
│                            │  - postgres-cluster-superuser   │  │
│                            │  - postgres-cluster-app         │  │
│                            └─────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
                                            │
                                            │ backups to
                                            ▼
                            ┌─────────────────────────────────┐
                            │  Cloudflare R2                  │
                            │  s3://homelab-postgres-backups  │
                            └─────────────────────────────────┘
```

## Connection Information

### Internal DNS Endpoints

| Endpoint | DNS Name | Port |
|----------|----------|------|
| Read-Write | postgres-cluster-rw.postgres.svc.cluster.local | 5432 |
| Read-Only | postgres-cluster-r.postgres.svc.cluster.local | 5432 |

### Connection String Format

```
postgresql://<user>:<password>@postgres-cluster-rw.postgres.svc.cluster.local:5432/<database>
```

## State Transitions

### Cluster States

```
Creating → Running → Upgrading → Running
    │         │          │
    └─────────┴──────────┴───────▶ Failed
```

| State | Description |
|-------|-------------|
| Creating | Initial deployment, PVC provisioning |
| Running | Healthy, accepting connections |
| Upgrading | Version or config update in progress |
| Failed | Error state, requires intervention |

### Backup States

```
Pending → Running → Completed
             │
             └───────▶ Failed
```

| State | Description |
|-------|-------------|
| Pending | Scheduled, waiting for execution |
| Running | Backup in progress |
| Completed | Successfully stored in R2 |
| Failed | Backup error |
