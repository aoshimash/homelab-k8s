# Quickstart: Migrate Home Assistant DB to PostgreSQL Cluster

**Feature**: 013-ha-postgres-migration
**Date**: 2026-01-09

## Prerequisites

- [ ] Flux is reconciling `k8s/` successfully
- [ ] CloudNativePG cluster `postgres-cluster` is Ready
- [ ] Home Assistant is running in namespace `home-assistant`
- [ ] Planned downtime window available (≤ 30 minutes)
- [ ] Rollback plan understood (retain SQLite for ≥ 24 hours)

## High-level Steps

1. **Prepare PostgreSQL database/user** (see `runbook-db-setup.md`)
   - Create database `homeassistant` and user `homeassistant`
   - Update `secret-db-url.sops.yaml` with actual password
   - Encrypt Secret with SOPS: `sops -e -i k8s/apps/home-assistant/app/secret-db-url.sops.yaml`

2. **Prepare configuration** (already done via GitOps)
   - Secret `home-assistant-db-url` (SOPS-encrypted)
   - ConfigMap `home-assistant-recorder-config` with Recorder settings
   - Deployment updated to inject `DB_URL` env var and mount ConfigMap

3. **Take pre-cutover backups** (see `runbook-backup.md`)
   - Longhorn snapshot of `home-assistant-config` PVC
   - CloudNativePG on-demand backup of `postgres-cluster`

4. **Execute cutover** (see `runbook-cutover.md`)
   - Stop Home Assistant: `kubectl scale deployment home-assistant -n home-assistant --replicas=0`
   - Run migration Job: `kubectl apply -f k8s/apps/home-assistant/app/job-db-migrate.yaml`
   - Wait for Job completion: `kubectl wait --for=condition=complete job/home-assistant-db-migrate -n home-assistant`
   - Start Home Assistant: `kubectl scale deployment home-assistant -n home-assistant --replicas=1`

5. **Verify continuity**
   - Check Home Assistant UI shows pre-migration history
   - Verify new events are recorded
   - Check logs for Recorder errors

6. **Observe for 24 hours**
   - Monitor logs for DB connectivity issues
   - Keep rollback option available (SQLite retained in PVC)
   - After 24h: Remove `job-db-migrate.yaml` from kustomization

## Suggested Verification Signals

```bash
# Home Assistant health
kubectl get deploy,pod -n home-assistant
kubectl logs -n home-assistant deploy/home-assistant --tail=200

# PostgreSQL cluster health
kubectl get cluster -n postgres
kubectl get pods -n postgres -l cnpg.io/cluster=postgres-cluster
```

## Notes

- This repo enforces GitOps: persistent changes should be applied via Git commits and Flux reconciliation.
- Database content operations (creating DB/user, importing data) are operational steps; keep a written log of actions taken and timestamps during cutover.
