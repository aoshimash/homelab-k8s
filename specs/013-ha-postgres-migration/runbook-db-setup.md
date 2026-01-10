# Runbook: PostgreSQL Database and User Setup for Home Assistant

**Feature**: 013-ha-postgres-migration  
**Date**: 2026-01-09

## Purpose

Create PostgreSQL database and user for Home Assistant Recorder before migration cutover.

## Prerequisites

- PostgreSQL cluster `postgres-cluster` is Ready in namespace `postgres`
- Access to PostgreSQL pod (for superuser access via `kubectl exec`)
- OR access to superuser secret `postgres-cluster-superuser` (if exists)
- OR use CloudNativePG ManagedRole for declarative user management

## Steps

### 1. Get PostgreSQL Credentials

```bash
# Get app user password (default application user)
kubectl get secret -n postgres postgres-cluster-app -o jsonpath='{.data.password}' | base64 -d

# Get connection string
kubectl get secret -n postgres postgres-cluster-app -o jsonpath='{.data.uri}' | base64 -d

# Note: If postgres-cluster-superuser secret exists, use that instead for admin operations
# Check available secrets: kubectl get secrets -n postgres | grep postgres-cluster
```

### 2. Connect to PostgreSQL

**Option A: Direct connection to PostgreSQL pod (Recommended)**

Connect directly to the PostgreSQL pod as the `postgres` superuser:

```bash
# Get PostgreSQL pod name
POD_NAME=$(kubectl get pods -n postgres -l cnpg.io/cluster=postgres-cluster -o jsonpath='{.items[0].metadata.name}')

# Connect to PostgreSQL pod and execute psql
kubectl exec -it -n postgres $POD_NAME -- psql -U postgres
```

**Option B: Use app user via service (if superuser access not needed)**

```bash
# Use app user (limited privileges - cannot create databases)
kubectl run -it --rm psql --image=postgres:16 --restart=Never -- \
  psql "postgresql://app:$(kubectl get secret -n postgres postgres-cluster-app -o jsonpath='{.data.password}' | base64 -d)@postgres-cluster-rw.postgres.svc.cluster.local:5432/postgres"
```

**Option C: If superuser secret exists**

```bash
# If postgres-cluster-superuser secret exists, use that
kubectl run -it --rm psql --image=postgres:16 --restart=Never -- \
  psql "postgresql://postgres:$(kubectl get secret -n postgres postgres-cluster-superuser -o jsonpath='{.data.password}' | base64 -d)@postgres-cluster-rw.postgres.svc.cluster.local:5432/postgres"
```

### 3. Generate Secure Password

Generate a strong password for the `homeassistant` user:

```bash
# Generate 32-character random password
openssl rand -base64 32 | tr -d "=+/" | cut -c1-32

# Save the password securely (you'll need it for step 4)
# Example output: tFId4FctBZEqw748Ldfj4TPwMTLlxVL3
```

**Note**: Save this password securely - you'll need it to update the Secret file.

### 4. Create Database and User

**Option A: Non-interactive commands (Recommended for automation)**

Execute commands directly without interactive session:

```bash
# Get PostgreSQL pod name
POD_NAME=$(kubectl get pods -n postgres -l cnpg.io/cluster=postgres-cluster -o jsonpath='{.items[0].metadata.name}')

# Set password variable (replace with generated password from step 3)
HA_PASSWORD="CHANGE_ME_TO_SECURE_PASSWORD"

# Create database
kubectl exec -n postgres $POD_NAME -- psql -U postgres -c "CREATE DATABASE homeassistant WITH ENCODING 'UTF8';"

# Create user with password
kubectl exec -n postgres $POD_NAME -- psql -U postgres -c "CREATE USER homeassistant WITH ENCRYPTED PASSWORD '$HA_PASSWORD';"

# Grant database privileges
kubectl exec -n postgres $POD_NAME -- psql -U postgres -c "GRANT ALL PRIVILEGES ON DATABASE homeassistant TO homeassistant;"

# Grant schema privileges
kubectl exec -n postgres $POD_NAME -- psql -U postgres -d homeassistant -c "GRANT ALL ON SCHEMA public TO homeassistant;"

# Verify creation
kubectl exec -n postgres $POD_NAME -- psql -U postgres -c "\l homeassistant"
kubectl exec -n postgres $POD_NAME -- psql -U postgres -c "\du homeassistant"
```

**Option B: Interactive session**

If you prefer an interactive session:

```bash
# Get PostgreSQL pod name
POD_NAME=$(kubectl get pods -n postgres -l cnpg.io/cluster=postgres-cluster -o jsonpath='{.items[0].metadata.name}')

# Connect interactively
kubectl exec -it -n postgres $POD_NAME -- psql -U postgres
```

Then execute SQL commands:

```sql
-- Create database for Home Assistant
CREATE DATABASE homeassistant WITH ENCODING 'UTF8';

-- Create user for Home Assistant (replace password with generated one)
CREATE USER homeassistant WITH ENCRYPTED PASSWORD 'CHANGE_ME_TO_SECURE_PASSWORD';

-- Grant privileges
GRANT ALL PRIVILEGES ON DATABASE homeassistant TO homeassistant;

-- Connect to homeassistant database and grant schema privileges
\c homeassistant
GRANT ALL ON SCHEMA public TO homeassistant;
```

**Alternative: Using CloudNativePG ManagedRole (Declarative)**

If you prefer declarative management, add to `k8s/configs/postgres/cluster.yaml`:

```yaml
spec:
  managed:
    roles:
      - name: homeassistant
        ensure: present
        login: true
        superuser: false
        createdb: false
        createrole: false
        connectionLimit: -1
        comment: "Home Assistant Recorder user"
```

Then create the database via SQL (still requires superuser access):

```sql
CREATE DATABASE homeassistant WITH ENCODING 'UTF8' OWNER homeassistant;
```

**Note**: ManagedRole creates the user, but database creation still requires SQL execution as superuser.

### 5. Update Secret with Actual Password

After creating the user, update `k8s/apps/home-assistant/app/secret-db-url.sops.yaml`:

1. Replace `CHANGE_ME` in `DB_URL` with the actual password you generated in step 3
2. Update the connection string format:
   ```
   postgresql://homeassistant:YOUR_PASSWORD@postgres-cluster-rw.postgres.svc.cluster.local:5432/homeassistant
   ```
3. Encrypt with SOPS (if Age key file is in project root):
   ```bash
   # Set Age key file location (if not already set)
   export SOPS_AGE_KEY_FILE=$(pwd)/age.agekey
   
   # Encrypt the Secret file
   sops -e -i k8s/apps/home-assistant/app/secret-db-url.sops.yaml
   ```
   
   **Note**: If `age.agekey` is in a different location, adjust the path accordingly.
   
4. Verify encryption:
   ```bash
   # Should show ENC[...] patterns, not plaintext
   grep -A 2 "DB_URL:" k8s/apps/home-assistant/app/secret-db-url.sops.yaml
   ```

### 6. Verify Connection

Test connection from within cluster:

**Option A: Using decrypted Secret (Recommended)**

```bash
# Set Age key file location (if not already set)
export SOPS_AGE_KEY_FILE=$(pwd)/age.agekey

# Extract DB_URL from decrypted Secret
DB_URL=$(sops -d k8s/apps/home-assistant/app/secret-db-url.sops.yaml | grep "DB_URL:" | sed 's/.*DB_URL: //')

# Test connection (non-interactive)
kubectl run test-ha-db-connection --image=postgres:16 --restart=Never --rm -- \
  psql "$DB_URL" -c "SELECT version(), current_database(), current_user;"

# Check logs for results
kubectl logs test-ha-db-connection 2>&1 | tail -5

# Clean up (if pod still exists)
kubectl delete pod test-ha-db-connection 2>&1 | grep -v "not found" || true
```

**Option B: Direct test with password**

```bash
# Replace YOUR_PASSWORD with actual password
kubectl run test-ha-db-connection --image=postgres:16 --restart=Never --rm -- \
  psql "postgresql://homeassistant:YOUR_PASSWORD@postgres-cluster-rw.postgres.svc.cluster.local:5432/homeassistant" \
  -c "SELECT version(), current_database(), current_user;"

# Check logs
kubectl logs test-ha-db-connection 2>&1 | tail -5
```

**Option C: Test permissions (CREATE/INSERT/DROP)**

```bash
export SOPS_AGE_KEY_FILE=$(pwd)/age.agekey
DB_URL=$(sops -d k8s/apps/home-assistant/app/secret-db-url.sops.yaml | grep "DB_URL:" | sed 's/.*DB_URL: //')

kubectl run test-ha-db-permissions --image=postgres:16 --restart=Never --rm -- \
  psql "$DB_URL" -c "CREATE TABLE IF NOT EXISTS test_permissions (id SERIAL PRIMARY KEY, test TEXT); INSERT INTO test_permissions (test) VALUES ('test'); SELECT COUNT(*) as row_count FROM test_permissions; DROP TABLE test_permissions;"

# Check logs
kubectl logs test-ha-db-permissions 2>&1 | tail -5
```

**Expected Results**:
- PostgreSQL version information should be displayed
- `current_database` should show `homeassistant`
- `current_user` should show `homeassistant`
- Permission test should show `row_count: 1` and complete without errors

## Notes

- Use a strong, unique password for the `homeassistant` user (32+ characters recommended)
- Document the password in a secure location (password manager) as it will be encrypted in Git
- The database will be created empty; Home Assistant will create tables on first connection
- If `age.agekey` is not in the project root, set `SOPS_AGE_KEY_FILE` environment variable to the correct path
- Non-interactive commands (Option A in step 4) are recommended for automation and reproducibility
- After successful verification, you can proceed to the cutover runbook (`runbook-cutover.md`)
