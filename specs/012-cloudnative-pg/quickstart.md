# Quickstart: CloudNativePG PostgreSQL Cluster

**Feature**: 012-cloudnative-pg  
**Date**: 2026-01-07

## Prerequisites

- Kubernetes cluster with Flux CD running
- Longhorn storage system deployed
- SOPS Age key configured in cluster (`sops-age` secret)
- Cloudflare R2 bucket created for backups

## Deployment Steps

### 1. Deploy Operator (Infrastructure)

The operator is deployed via Flux from `k8s/infrastructure/cloudnative-pg/`.

```bash
# Verify operator deployment
kubectl get deploy -n cnpg-system
# Expected: cnpg-controller-manager 1/1 Ready

# Verify CRDs installed
kubectl get crd | grep cnpg
# Expected: clusters.postgresql.cnpg.io, backups.postgresql.cnpg.io, etc.
```

### 2. Deploy PostgreSQL Cluster (Configs)

The cluster is deployed via Flux from `k8s/configs/postgres/`.

```bash
# Verify cluster status
kubectl get cluster -n postgres
# Expected: postgres-cluster with STATUS "Cluster in healthy state"

# Verify pod running
kubectl get pods -n postgres
# Expected: postgres-cluster-1 Running

# Verify services
kubectl get svc -n postgres
# Expected: postgres-cluster-rw, postgres-cluster-r, postgres-cluster-ro
```

### 3. Get Superuser Credentials

```bash
# Get superuser password
kubectl get secret -n postgres postgres-cluster-superuser -o jsonpath='{.data.password}' | base64 -d

# Get connection string
kubectl get secret -n postgres postgres-cluster-superuser -o jsonpath='{.data.uri}' | base64 -d
```

## Connecting to PostgreSQL

### From Within Cluster

```bash
# Start a psql pod
kubectl run -it --rm psql --image=postgres:16 --restart=Never -- \
  psql "postgresql://postgres:$(kubectl get secret -n postgres postgres-cluster-superuser -o jsonpath='{.data.password}' | base64 -d)@postgres-cluster-rw.postgres.svc.cluster.local:5432/postgres"
```

### Connection String for Applications

```
postgresql://<user>:<password>@postgres-cluster-rw.postgres.svc.cluster.local:5432/<database>
```

## Creating Application Database/User

```sql
-- Connect as superuser first
-- Create database for application
CREATE DATABASE myapp;

-- Create user for application
CREATE USER myapp_user WITH ENCRYPTED PASSWORD 'secure_password';

-- Grant privileges
GRANT ALL PRIVILEGES ON DATABASE myapp TO myapp_user;

-- Connect to myapp database and grant schema privileges
\c myapp
GRANT ALL ON SCHEMA public TO myapp_user;
```

## Backup Operations

### Manual Backup

```bash
# Create on-demand backup
kubectl apply -f - <<EOF
apiVersion: postgresql.cnpg.io/v1
kind: Backup
metadata:
  name: manual-backup-$(date +%Y%m%d-%H%M%S)
  namespace: postgres
spec:
  cluster:
    name: postgres-cluster
EOF

# Check backup status
kubectl get backup -n postgres
```

### List Backups

```bash
kubectl get backup -n postgres
```

### Restore from Backup

```bash
# Create new cluster from backup
kubectl apply -f - <<EOF
apiVersion: postgresql.cnpg.io/v1
kind: Cluster
metadata:
  name: postgres-cluster-restored
  namespace: postgres
spec:
  instances: 1
  storage:
    size: 10Gi
    storageClass: longhorn
  bootstrap:
    recovery:
      backup:
        name: <backup-name>
EOF
```

## Verification Commands

```bash
# Full health check
echo "=== Operator ===" && \
kubectl get deploy -n cnpg-system && \
echo "=== Cluster ===" && \
kubectl get cluster -n postgres && \
echo "=== Pods ===" && \
kubectl get pods -n postgres && \
echo "=== Services ===" && \
kubectl get svc -n postgres && \
echo "=== Backups ===" && \
kubectl get backup -n postgres && \
echo "=== ScheduledBackups ===" && \
kubectl get scheduledbackup -n postgres
```

## Troubleshooting

### Cluster Not Ready

```bash
# Check cluster events
kubectl describe cluster -n postgres postgres-cluster

# Check operator logs
kubectl logs -n cnpg-system deploy/cnpg-controller-manager
```

### Pod Not Starting

```bash
# Check pod events
kubectl describe pod -n postgres -l cnpg.io/cluster=postgres-cluster

# Check PVC status
kubectl get pvc -n postgres
```

### Backup Failed

```bash
# Check backup details
kubectl describe backup -n postgres <backup-name>

# Verify R2 credentials
kubectl get secret -n postgres postgres-r2-credentials
```

## Next Steps

1. Create application-specific databases and users
2. Configure applications to use PostgreSQL connection string
3. Set up monitoring dashboards (Grafana)
4. Test backup/restore procedure
