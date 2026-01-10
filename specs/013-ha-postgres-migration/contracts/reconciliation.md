# Reconciliation Contract: Home Assistant PostgreSQL Migration

**Feature**: 013-ha-postgres-migration
**Date**: 2026-01-09

## Overview

This contract defines the expected reconciliation behavior for migrating Home Assistant’s database configuration from SQLite to the shared PostgreSQL cluster, under Flux GitOps control.

## Flux Kustomization Hierarchy

```
k8s/flux/apps-kustomization.yaml
    └── k8s/apps/kustomization.yaml
        └── k8s/apps/home-assistant/kustomization.yaml
            ├── namespace.yaml
            └── app/kustomization.yaml
                ├── deployment.yaml
                ├── pvc.yaml
                ├── service.yaml
                ├── ingress.yaml
                ├── configmap.yaml
                ├── secret-db-url.sops.yaml      # new
                └── (optional) migration job     # temporary during cutover
```

## Reconciliation Scenarios

### Scenario 1: Configure Home Assistant to use PostgreSQL (steady state)

**Trigger**: Commit that adds/updates:
- SOPS-encrypted Secret containing `DB_URL`
- Home Assistant configuration enabling Recorder `db_url: !env_var DB_URL`
- Deployment env var injection from the Secret

**Expected Behavior**:
1. Flux decrypts SOPS Secret and applies it in `home-assistant` namespace.
2. Deployment updates roll out normally.
3. Home Assistant starts and connects to PostgreSQL; history continues to be written.

**Verification**:
```bash
kubectl get secret -n home-assistant home-assistant-db-url
kubectl rollout status deployment/home-assistant -n home-assistant
kubectl logs -n home-assistant deploy/home-assistant --tail=200
```

**Success Signals**:
- Pod is Running and Ready
- No persistent Recorder DB connection errors in logs
- UI shows historical charts before/after cutover (per spec)

---

### Scenario 2: Temporary migration Job (cutover-only resource)

**Trigger**: Commit that temporarily introduces a Kubernetes Job manifest for the SQLite → PostgreSQL copy.

**Expected Behavior**:
1. Flux applies the Job manifest.
2. The Job runs to completion exactly once (Kubernetes Job semantics).
3. After success, a follow-up commit removes the Job manifest (prune cleans it up).

**Verification**:
```bash
kubectl get job -n home-assistant
kubectl logs -n home-assistant job/<job-name> --tail=200
kubectl get pods -n home-assistant -l job-name=<job-name>
```

**Success Signals**:
- Job `status.succeeded` becomes `1`
- No CrashLoopBackOff in the migration pod

---

### Scenario 3: Rollback via Git revert (within 24h window)

**Trigger**: Revert the cutover commit(s) that switch Home Assistant to PostgreSQL.

**Expected Behavior**:
1. Flux reconciles the manifests back to the SQLite configuration.
2. Home Assistant starts and functions using the pre-migration SQLite DB (or its restored snapshot).

**Verification**:
```bash
git log --oneline -5
flux get kustomizations
kubectl rollout status deployment/home-assistant -n home-assistant
```

## Health Checks

| Check | Method | Expected |
|-------|--------|----------|
| Pod Running | `kubectl get pod -n home-assistant` | 1/1 Running |
| Readiness | `kubectl get pod -n home-assistant` | Ready = True |
| Recorder DB connectivity | Logs | No sustained connection failures |
| PostgreSQL service resolution | In-cluster DNS test | Service resolves and is reachable |

## Notes

- Database *content* changes (creating the HA database/user, importing data) are operational steps and cannot be fully expressed as immutable Git resources. The GitOps contract here focuses on Kubernetes state reconciliation and auditability.
