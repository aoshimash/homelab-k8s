# Feature Specification: Introduce Longhorn with R2 Backups

**Feature Branch**: `002-longhorn-r2-backup`
**Created**: 2026-01-02
**Status**: Draft
**Input**: User description: "longhornを導入。現在はシングルノードであることに注意し、レプリカ数などを調整する。Backup TargetはCloudflare R2にする。Secret管理はSOPS＋ageを使いFluxで復号化する。"

## Clarifications

### Session 2026-01-02

- Q: What should be the default replica count for new volumes in a single-node cluster? → A: Default replicas = 1.
- Q: What backup schedule should be used for volumes? → A: Automatic daily backups.
- Q: Should backups be skipped when unchanged? → A: Run, but minimal overhead if unchanged.
- Q: What backup retention should be used? → A: Keep 30 days.

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Reliable single-node persistent storage (Priority: P1)

As a cluster operator, I want Longhorn to provide reliable persistent storage on a single-node cluster so that stateful workloads can run without manual storage operations and without being degraded purely due to single-node topology (default replica count = 1).

**Why this priority**: Persistent storage is foundational for running stateful services; on a single node, defaults that assume multiple replicas can cause unnecessary “degraded” states or scheduling failures.

**Independent Test**: After reconciliation, a new workload can request storage and successfully read/write data using the default storage behavior without operator intervention.

**Acceptance Scenarios**:

1. **Given** a single-node cluster with GitOps reconciliation enabled, **When** Longhorn is reconciled from the repository state, **Then** Longhorn is healthy and ready for provisioning volumes on the single node.
2. **Given** a test workload that requests a new persistent volume, **When** it writes and then reads data, **Then** the data is preserved across pod restarts.

---

### User Story 2 - Backup and restore volumes to Cloudflare R2 (Priority: P2)

As a cluster operator, I want to back up and restore Longhorn volumes to Cloudflare R2 so that I can recover data after accidental deletion, corruption, or node/cluster rebuild.

**Why this priority**: Backups provide operational safety and disaster recovery; object storage off-cluster reduces risk from single-node failures.

**Independent Test**: A sample volume can be backed up to R2, deleted, and restored with data integrity verified.

**Acceptance Scenarios**:

1. **Given** a volume containing known test data, **When** a backup is created and stored in Cloudflare R2, **Then** the backup completes successfully and is listed as available for restore.
2. **Given** an existing backup in Cloudflare R2, **When** a restore is executed into a new volume, **Then** the restored volume contains the expected data and can be mounted by a workload.

---

### User Story 3 - Encrypted backup credentials managed in Git (Priority: P3)

As a cluster operator, I want backup credentials and configuration to be stored encrypted in Git and decrypted only at reconciliation time so that secrets are never committed in plaintext while remaining fully reproducible through GitOps.

**Why this priority**: Secure secret handling is mandatory for object storage credentials; GitOps requires the desired state to live in Git without leaking secrets.

**Independent Test**: The repository contains only encrypted secret material, and the cluster reconciles successfully with usable credentials for backups.

**Acceptance Scenarios**:

1. **Given** the repository contains the required backup credential secrets in encrypted form, **When** Flux reconciles the cluster state, **Then** the decrypted secrets exist in the cluster and Longhorn can authenticate to Cloudflare R2 for backup operations.
2. **Given** a user scans the Git repository contents, **When** they search for object storage access keys, **Then** no plaintext credentials are present.

---

### Edge Cases

- What happens when the cluster remains single-node but the configured volume replication expectations assume multiple nodes?
- What happens when the cluster later becomes multi-node but the default replica count remains set to 1 (availability expectations vs. operational simplicity)?
- What happens when Cloudflare R2 is temporarily unreachable or returns authentication errors?
- How does the system behave when a backup completes but a restore is attempted to a cluster with different node identity or storage layout?
- What happens when encryption keys for decryption are missing or rotated incorrectly during reconciliation?
- How does the system prevent partial/invalid configuration from leaving backups in an unknown state?

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: The system MUST install and operate Longhorn as the cluster’s persistent storage provider under a GitOps workflow.
- **FR-002**: The system MUST be configured for a single-node cluster such that volumes are not degraded solely because additional nodes are unavailable.
- **FR-002a**: The default replica count for new volumes MUST be 1 while the cluster is single-node.
- **FR-003**: The system MUST allow stateful workloads to dynamically provision persistent volumes and retain data across pod restarts.
- **FR-004**: The system MUST support creating backups of Longhorn volumes to an off-cluster object storage target.
- **FR-005**: The system MUST use Cloudflare R2 as the backup target for Longhorn volume backups.
- **FR-006**: The system MUST support restoring a volume from a backup stored in Cloudflare R2 into a usable volume for workloads.
- **FR-007**: The system MUST store all backup credentials/configuration secrets encrypted at rest in Git using SOPS with age.
- **FR-008**: The system MUST decrypt encrypted secrets during reconciliation via Flux so that plaintext secrets are never stored in Git.
- **FR-009**: The system MUST provide operator-visible signals (events/status) for backup success/failure and restore success/failure.
- **FR-010**: The system MUST support automatic daily backups for volumes.
- **FR-011**: When a scheduled backup runs with no meaningful data changes, the system SHOULD still be able to run successfully while minimizing additional storage consumption and completion time (i.e., avoid full re-copying of unchanged data).
- **FR-012**: The system MUST retain volume backups for 30 days.

### Key Entities *(include if feature involves data)*

- **Longhorn Volume**: A persistent storage unit used by workloads; has lifecycle state, size, and attachment/mount usage.
- **Backup Target**: The external destination where backups are stored (Cloudflare R2 bucket/namespace).
- **Backup Credential**: Authentication material enabling access to the Backup Target (kept encrypted in Git).
- **Volume Backup**: A recoverable point-in-time artifact stored in the Backup Target.
- **Volume Restore**: An operation that recreates a volume from a Volume Backup for workload consumption.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: In a single-node cluster, a newly created persistent volume can be provisioned and mounted by a test workload within 5 minutes of requesting it.
- **SC-002**: A test workload can write data to a volume, be restarted, and read the same data successfully with a 100% pass rate across 5 consecutive runs.
- **SC-003**: A backup of a test volume to Cloudflare R2 completes successfully at least 3 times in a row, and each backup is available for restore.
- **SC-004**: Restoring from a Cloudflare R2 backup reproduces expected test data with a 100% integrity check pass rate across 3 restore attempts.
- **SC-005**: The Git repository contains 0 plaintext instances of object storage credentials (as verified by keyword search for access key patterns) at all times.
- **SC-006**: Daily scheduled backups are observed to run successfully for a test volume for 7 consecutive days (or 7 consecutive scheduled executions), with 0 manual triggers required.
- **SC-007**: The backup retention policy is configured so that backups older than 30 days are not retained while backups within the last 30 days remain available for restore.
