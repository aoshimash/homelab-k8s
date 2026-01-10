# Contract: Home Assistant DB migration backup & restore

**Feature**: 013-ha-postgres-migration
**Date**: 2026-01-09

## Purpose

Define the minimum expected behavior for backups, rollback, and restores when migrating Home Assistant from SQLite to PostgreSQL.

## Preconditions

- Home Assistant PVC (`home-assistant-config`) is healthy and restorable via Longhorn.
- CloudNativePG PostgreSQL cluster is Ready and backups are functioning.
- A planned downtime window (≤ 30 minutes) is scheduled for cutover.

## Backup Contract (pre-cutover)

- A pre-cutover backup exists for the SQLite source (at least one of):
  - Longhorn snapshot/backup of `home-assistant-config` volume, or
  - A copy of `home-assistant_v2.db` stored outside the running pod (operator-managed artifact).
- A pre-cutover PostgreSQL backup is triggered (on-demand) to establish a baseline restore point.
- Backup failures are operator-visible with actionable error details.

## Restore / Rollback Contract (within 24 hours)

- Rollback to SQLite remains possible for **≥ 24 hours** after cutover by retaining the pre-migration SQLite state.
- If rollback is executed:
  - Home Assistant returns to a working state using SQLite within **≤ 30 minutes** of the decision to rollback.
  - Post-cutover changes to PostgreSQL may be lost (explicitly acceptable; spec prioritizes service restoration).

## PostgreSQL Restore Contract

- A PostgreSQL restore from an available CloudNativePG backup produces a usable cluster (or restored cluster) that Home Assistant can connect to.
- Restored data integrity is validated by:
  - Home Assistant UI history/logbook continuity checks, and
  - Basic sanity queries (row counts / recent timestamps), as documented in the runbook.

## Mapping to Success Criteria

- **SC-001**: Planned downtime ≤ 30 minutes
- **SC-003**: 24h stable operation post-migration (rollback available during this period)
- **SC-004**: Rollback restores service within ≤ 30 minutes
- **SC-005**: Rollback remains available for ≥ 24 hours

## Verification Signals

- Longhorn: volume backup/snapshot is listed and restorable.
- CloudNativePG: `Backup` objects complete successfully; restore procedure is documented and repeatable.
- Rollback dry-run in a test environment (recommended) demonstrates feasibility within time bounds.

## Restore Procedure

See `runbook-restore.md` for detailed PostgreSQL backup restore and validation procedure.
