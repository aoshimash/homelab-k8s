# Feature Specification: Migrate Home Assistant DB to PostgreSQL Cluster

**Feature Branch**: `013-ha-postgres-migration`
**Created**: 2026-01-09
**Status**: Draft
**Input**: User description: "Home AssistantのSQLiteをPostgress-clusterに移行する"

## Clarifications

### Session 2026-01-09

- Q: What history range should be migrated? → A: Migrate all existing history (no intentional truncation).
- Q: What cutover approach should be used? → A: Planned downtime (stop Home Assistant during cutover).
- Q: How long should rollback remain possible after cutover? → A: 24 hours.
- Q: What should happen if the database becomes unavailable after migration? → A: Keep Home Assistant running; history recording may pause until the database recovers.
- Q: What is the maximum acceptable planned downtime? → A: ≤ 30 minutes.

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Preserve history after migration (Priority: P1)

As a homelab operator, I want Home Assistant to use the PostgreSQL cluster as its primary database instead of the local SQLite database so that history and state-related data is more durable and survives restarts, upgrades, and single-node failures.

**Why this priority**: Durable history/state data is essential for dashboards, troubleshooting, and long-term insights. Losing it undermines the value of Home Assistant.

**Independent Test**: Migrate a non-production copy (or a snapshot) and verify that pre-migration history is visible and that new events continue to be recorded after migration.

**Acceptance Scenarios**:

1. **Given** an existing Home Assistant instance with historical data stored in SQLite, **When** the migration is completed and Home Assistant is started, **Then** the UI loads normally and historical charts show pre-migration data.
2. **Given** the migrated instance is running, **When** new sensor updates and events occur, **Then** new history entries appear and still exist after a Home Assistant restart.

---

### User Story 2 - Safe rollback on migration failure (Priority: P2)

As a homelab operator, I want a clear rollback path so that if the migration fails, I can restore service quickly without permanently losing the original data.

**Why this priority**: Database migrations are inherently risky. A rollback plan prevents extended downtime and reduces the chance of irreversible data loss.

**Independent Test**: Intentionally fail the migration in a test environment and confirm Home Assistant can be restored to the pre-migration state within the defined time window.

**Acceptance Scenarios**:

1. **Given** a migration attempt fails before completion, **When** rollback is executed, **Then** Home Assistant starts successfully using the pre-migration database and core features work (dashboards, automations, basic device state updates).
2. **Given** rollback is completed, **When** Home Assistant runs for 1 hour, **Then** no user-visible errors related to data storage prevent normal operation.

---

### User Story 3 - Restore from backup and verify data integrity (Priority: P3)

As a homelab operator, I want to be able to restore the Home Assistant database from backup so that disaster recovery is proven (not theoretical).

**Why this priority**: Migration is a good opportunity to validate backups and recovery. Recovery confidence is critical for maintenance and upgrades.

**Independent Test**: Restore a backup into a test Home Assistant instance and confirm expected data (history/state) is present.

**Acceptance Scenarios**:

1. **Given** a valid database backup exists, **When** it is restored into a test instance, **Then** historical data is visible and consistent with expectations.

---

### Edge Cases

- Migration takes longer than expected due to large history volume.
- Migration is interrupted (power loss / node restart) part-way through.
- The PostgreSQL cluster is temporarily unavailable during cutover.
- The PostgreSQL cluster becomes unavailable after cutover (Home Assistant should remain usable; history may pause).
- Credentials/permissions for the target database are incorrect.
- Partial migration results in missing or duplicated history data.
- Home Assistant starts but experiences degraded behavior due to database connectivity issues.

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: The system MUST migrate Home Assistant’s existing database content from SQLite to the PostgreSQL cluster.
- **FR-002**: The system MUST preserve **all** existing history/state-related data present in SQLite at the time of migration (no intentional truncation).
- **FR-003**: The system MUST ensure Home Assistant uses the PostgreSQL cluster as the active database after migration (i.e., new history is written to the new database).
- **FR-004**: The system MUST define a bounded maintenance window for the migration and communicate it in the spec (see Success Criteria).
- **FR-009**: The system MUST use a planned cutover approach where Home Assistant is stopped during the cutover window to prevent writes during migration.
- **FR-010**: If the database is temporarily unavailable after cutover, the system MUST prioritize keeping Home Assistant operational, acknowledging that history recording may pause until database connectivity is restored.
- **FR-011**: The planned maintenance window (service unavailability) MUST be **≤ 30 minutes**.
- **FR-005**: The system MUST provide a rollback procedure that restores Home Assistant to a working pre-migration state if migration fails.
- **FR-006**: The system MUST keep the pre-migration SQLite database available for rollback for **at least 24 hours** after cutover and validation.
- **FR-007**: The system MUST provide a verification step that confirms historical data continuity (pre- and post-migration) from a user perspective.
- **FR-008**: The system MUST support backup and restore validation for the migrated database (at least in a test environment).

### Assumptions & Dependencies

- **A-001 (Assumption)**: A recent, restorable backup of the pre-migration SQLite database is available before starting the migration.
- **A-002 (Assumption)**: The data volume is within a range that allows completion within the maintenance window stated in Success Criteria.
- **D-001 (Dependency)**: The PostgreSQL cluster is reachable from Home Assistant during and after cutover.
- **D-002 (Dependency)**: Database credentials and permissions required for normal operation are available and validated before cutover.

### Out of Scope

- Migrating or redesigning Home Assistant configuration files, automations, or integrations that are not stored in the database.
- Changing retention policies or purging historical data as part of this feature (data should be preserved as-is per FR-002).

### Key Entities *(include if feature involves data)*

- **Home Assistant Database**: The persistent store for Home Assistant’s history/state-related information used for dashboards and troubleshooting.
- **History Entry**: A record representing a change/event over time that should remain visible in charts and logbook views.
- **Migration Run**: A single execution of the migration process with a clear start/end, outcome (success/failure), and verification result.
- **Backup Snapshot**: A point-in-time capture of the database state used for rollback and disaster recovery.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: Home Assistant service unavailability during the migration is **≤ 30 minutes** (planned maintenance window).
- **SC-002**: After migration, users can view historical charts/logbook entries from **before** the migration and see new entries created **after** the migration (continuity verified).
- **SC-003**: For **24 hours** after migration, Home Assistant operates normally without user-visible data-storage failures that prevent dashboards and device state updates.
- **SC-004**: If rollback is required, Home Assistant is restored to a working pre-migration state within **≤ 30 minutes** from the decision to rollback.
- **SC-005**: The rollback option remains available for **at least 24 hours** after cutover (pre-migration SQLite retained per FR-006).
