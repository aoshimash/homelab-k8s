# Research: Home Assistant SQLite → PostgreSQL Cluster Migration

**Feature**: 013-ha-postgres-migration
**Date**: 2026-01-09

## Research Topics

### 1. Home Assistant Recorder PostgreSQL configuration

**Decision**: Configure Home Assistant Recorder with:
- `recorder.db_url` pointing to PostgreSQL
- Connection retries (`db_max_retries`, `db_retry_wait`) to reduce impact of temporary DB unavailability

**Rationale**:
- Home Assistant officially supports PostgreSQL as a Recorder backend via `recorder.db_url`.
- Retry settings align with the desired behavior: keep the service usable when the DB is temporarily unavailable (history may pause).

**Alternatives Considered**:
- Keep SQLite: simplest but does not meet durability/operability goal.
- MariaDB: common recommendation, but the project already runs a PostgreSQL cluster.

### 2. Supplying credentials without committing secrets

**Decision**: Use `!env_var DB_URL` in `configuration.yaml` and inject `DB_URL` from a Kubernetes Secret encrypted with SOPS.

**Rationale**:
- Avoids embedding credentials in Git-tracked configuration files.
- Fits the repository’s Secrets Management principle (SOPS + Flux decryption).

**Alternatives Considered**:
- `!secret` via `secrets.yaml`: would require managing/merging an existing `secrets.yaml` in the PVC.
- Hardcoded `db_url`: rejected due to secret exposure risk.

### 3. Migration method (SQLite file → PostgreSQL)

**Decision**: Use `pgloader` executed as a one-time Kubernetes Job during cutover, reading the SQLite DB file from the Home Assistant PVC and writing to PostgreSQL.

**Rationale**:
- Commonly used approach for SQLite → PostgreSQL migrations.
- Can be run fully in-cluster with no node access, aligning with immutable infrastructure.
- Enables a predictable planned downtime window: stop Home Assistant, run one migration job, start Home Assistant.

**Alternatives Considered**:
- SQL dump/import: fragile due to type and schema differences and may require extensive manual fixes.
- Custom application-level migrator: higher maintenance and correctness risk for this repo.

**Known Risks / Mitigations**:
- SQLite may contain data that violates constraints enforced more strictly by PostgreSQL.
  - Mitigation: run a pre-flight dry run on a restored/snapshot copy when possible and use pgloader options to reduce constraint churn; validate with UI continuity checks.
- Sequence values may need adjustment after bulk import.
  - Mitigation: run post-migration sanity checks and adjust sequences if required (documented in quickstart/runbook).

### 4. Cutover and rollback strategy

**Decision**:
- Cutover with planned downtime (Home Assistant stopped during migration).
- Keep rollback possible for **≥ 24 hours** by retaining the pre-migration SQLite database and/or a restorable snapshot.

**Rationale**:
- Prevents writes during migration, ensuring consistent copy.
- 24-hour observation window aligns with the spec’s success criteria (stable operation for 24 hours).

**Alternatives Considered**:
- Live migration with no downtime: rejected due to complexity and risk.
- Immediate cleanup of SQLite after cutover: rejected due to rollback requirement.

## Summary of Key Decisions

| Topic | Decision | Key Reason |
|-------|----------|------------|
| Recorder backend | PostgreSQL via `recorder.db_url` | Durability and multi-tenant DB operations |
| DB credentials | `DB_URL` via `!env_var` + SOPS Secret | No secrets in Git; GitOps-compatible |
| Migration tool | pgloader Job | In-cluster, repeatable, common approach |
| Cutover | Planned downtime | Data consistency, simpler rollback |
| Rollback window | 24 hours | Matches post-cutover observation period |
