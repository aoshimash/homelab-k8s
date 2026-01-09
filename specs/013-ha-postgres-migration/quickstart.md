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

1. Prepare PostgreSQL database/user for Home Assistant
2. Prepare `DB_URL` secret (SOPS) and Home Assistant Recorder configuration
3. Take pre-cutover backups (SQLite state + PostgreSQL baseline)
4. Stop Home Assistant and run migration job (pgloader)
5. Start Home Assistant with PostgreSQL Recorder and verify continuity
6. Observe for 24 hours, then clean up temporary migration resources

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
