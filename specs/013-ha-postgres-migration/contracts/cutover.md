# Contract: Cutover runbook (SQLite → PostgreSQL)

**Feature**: 013-ha-postgres-migration  
**Date**: 2026-01-09

## Purpose

Define a repeatable, operator-friendly cutover sequence that:
- preserves all history,
- keeps downtime ≤ 30 minutes, and
- keeps rollback possible for ≥ 24 hours.

## Pre-flight Checklist (before downtime)

- PostgreSQL cluster is Ready (`postgres-cluster`).
- Target database and user exist for Home Assistant (least-privilege).
- `DB_URL` Secret is prepared (SOPS) and reviewed.
- Home Assistant configuration change is prepared (Recorder uses `!env_var DB_URL` + retry settings).
- Pre-cutover backups are created (SQLite state + PostgreSQL baseline backup).
- A test migration has been executed at least once on a non-production copy (recommended to validate downtime feasibility).

## Cutover Steps (downtime window)

1. **Stop Home Assistant**
   - Scale the Deployment to 0 to prevent writes during migration.
2. **Run migration Job**
   - Execute pgloader against the SQLite database file in the PVC and load into PostgreSQL.
3. **Switch Home Assistant to PostgreSQL**
   - Apply the configuration that enables Recorder `db_url` pointing to PostgreSQL via env var.
4. **Start Home Assistant**
   - Scale the Deployment back to 1.

## Verification (immediate)

- Home Assistant UI loads normally.
- Historical charts/logbook show pre-migration data.
- New events are recorded after restart.
- No sustained Recorder DB errors in logs.

## Post-cutover Observation (24 hours)

- Monitor logs for DB connectivity issues.
- Confirm history continues to accumulate.
- Keep rollback option available (retain SQLite state).

## Completion

After the 24h observation window passes:
- Remove temporary migration resources (Job manifests) from Git.
- Keep backups as per normal retention policies.
