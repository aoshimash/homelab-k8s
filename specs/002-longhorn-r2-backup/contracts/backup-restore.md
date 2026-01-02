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

## Verification signals

- Backup artifacts are listed as available for restore.
- Restore results in a mountable volume that passes a simple read/write integrity check.
