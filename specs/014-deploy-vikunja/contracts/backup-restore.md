# Contract: Backup & Restore (Vikunja)

**Feature**: 014-deploy-vikunja  
**Date**: 2026-01-10

## Scope

This contract defines what MUST be backed up and how restore validation is performed for Vikunja.

## What MUST Be Backed Up

### 1) PostgreSQL Database (authoritative task data)

- **Mechanism**: CloudNativePG backups to R2 using:
  - `Cluster.spec.backup.barmanObjectStore`
  - `ScheduledBackup` (daily schedule)
- **Coverage**: users, projects, tasks, sharing, and all metadata stored in the DB

### 2) File/Attachment Storage

- **Mechanism**: Longhorn volume backups (daily) enabled via PVC annotation:
  - `recurring-job-selector.longhorn.io: '[{"name":"backup-daily","isGroup":false}]'`
- **Coverage**: attachments and any file-based data stored on the Vikunja files volume

## Backup Success Signals

- CNPG ScheduledBackup runs successfully each day (as observed in CNPG status/events/logs).
- Longhorn shows daily backups for the Vikunja files volume.

## Restore Procedure (validation-oriented)

### Database Restore (CNPG)

**Goal**: Restore Vikunja DB to a known-good state from backups.

Validation requirements:
- Restored DB contains expected sample data (projects/tasks created during test window)
- Application can connect to the restored DB and sign-in works

### Files Restore (Longhorn)

**Goal**: Restore attachments/files volume from Longhorn backups.

Validation requirements:
- Restored volume is mountable by the workload
- Sample attachment file created during test window is present and accessible in UI

## Restore Test (minimum)

At least once per change cycle:
1. Create a test project + tasks.
2. Upload a test attachment (if enabled in the UI).
3. Confirm next scheduled backups complete (DB + files).
4. Perform a restore drill into a clean environment (or controlled restore), and verify:
   - Tasks/projects exist
   - Attachment exists

