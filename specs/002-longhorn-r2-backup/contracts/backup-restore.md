# Contract: Longhorn backup & restore to Cloudflare R2

## Purpose

Define the minimum expected behavior for backups and restores when using Cloudflare R2 as the backup target.

## Preconditions

- Longhorn is Ready and able to provision volumes.
- Backup target and credentials are configured and valid.
- Object storage bucket exists and is reachable from the cluster.

## Backup contract

- Backups are scheduled daily and execute without manual triggers.
- Backups are retained for 30 days.
- When data is unchanged, scheduled backups run successfully with minimal additional storage consumption and completion time (no full re-copying of unchanged data).
- Failures are operator-visible and include actionable error information.

## Restore contract

- A restore operation from an existing backup produces a usable volume.
- Restored data integrity matches the backup source (validated by test workload).
- Restore failures are operator-visible and include actionable error information.

## Mapping to success criteria

- **SC-003**: Backups can be created and stored in R2.
- **SC-004**: Restores can be performed successfully into a new volume.
- **SC-006**: Daily schedules run successfully.
- **SC-007**: Restore produces data integrity consistent with the source volume.

## Verification signals

- Backup artifacts are listed as available for restore.
- Restore results in a mountable volume that passes a simple read/write integrity check.

## Operator checklist (recommended)

- Confirm backup settings are configured:
  - Backup target points to Cloudflare R2.
  - Credential Secret exists and is readable by Longhorn.
- Confirm backups are running:
  - A daily backup job exists (recurring job or schedule) and reports success.
  - A new backup artifact appears in the UI/list and is restorable.
- Confirm restore workflow:
  - Restore into a new volume (do not overwrite the source).
  - Attach the restored volume to a test workload and validate the expected content.
