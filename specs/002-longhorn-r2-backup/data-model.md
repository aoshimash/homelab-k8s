# Data Model: Longhorn + R2 Backups

This document captures the conceptual entities used to reason about Longhorn volumes and backups for this feature. It is intentionally implementation-agnostic.

## Entities

### Cluster

- **clusterId**: Identifier used to scope configuration and (optionally) backup paths.
- **topology**: `single-node` (current) | `multi-node` (future).

### Node

- **nodeId**: Unique node identifier.
- **ready**: Whether the node is usable for scheduling.

### Longhorn Volume

- **volumeName**: Logical name/identifier.
- **size**: Allocated size.
- **replicaCount**: Desired replica count for durability (default = 1 for single-node).
- **state**: `detached` | `attached` | `faulted` | `unknown`.
- **health**: `healthy` | `degraded` | `faulted`.
- **workloadAttachment**: Optional association to a workload consuming the volume.

**Rules**
- In a `single-node` topology, `replicaCount` default MUST be 1.

### Backup Target (Cloudflare R2)

- **provider**: `S3-compatible`.
- **endpoint**: Account-scoped endpoint address.
- **bucketName**: Bucket storing backups.
- **pathPrefix**: Optional prefix to group backups (e.g., by cluster).
- **credentialsRef**: Reference to backup credentials.

### Backup Credential

- **accessKeyId**: Identifier for authentication (stored encrypted in Git).
- **secretAccessKey**: Secret for authentication (stored encrypted in Git).
- **scope**: Bound to `bucketName` (and optionally a prefix) by storage policy outside the spec.
- **rotationState**: `active` | `rotated` | `revoked`.

### Volume Backup

- **backupId**: Unique identifier for a backup artifact.
- **volumeRef**: Which volume it belongs to.
- **createdAt**: Timestamp.
- **status**: `pending` | `completed` | `failed`.
- **retentionWindowDays**: 30 (per spec).

### Backup Schedule

- **scheduleId**: Identifier for a schedule rule.
- **cadence**: `daily`.
- **enabled**: Boolean.
- **retentionWindowDays**: 30.
- **behaviorWhenUnchanged**: `run-with-minimal-overhead`.

### Volume Restore

- **restoreId**: Identifier for a restore operation.
- **sourceBackupRef**: Backup to restore from.
- **targetVolumeRef**: Volume created/restored into.
- **status**: `pending` | `completed` | `failed`.

## State Transitions (high level)

### Volume

- `detached` → `attached`: workload mounts/attaches volume.
- `attached` → `detached`: workload stops or is rescheduled.
- Any → `faulted`: underlying storage failure or unrecoverable error.

### Backup

- `pending` → `completed`: backup finishes successfully and is restorable.
- `pending` → `failed`: backup fails and provides operator-visible error signals.

### Restore

- `pending` → `completed`: restored volume becomes usable by workloads.
- `pending` → `failed`: restore fails and provides operator-visible error signals.
