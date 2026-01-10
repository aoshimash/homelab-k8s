# Operational Notes: Home Assistant PostgreSQL Migration

**Feature**: 013-ha-postgres-migration
**Date**: 2026-01-09

## Purpose

Document operational learnings, gotchas, and best practices discovered during migration implementation.

## Key Learnings

### SOPS Secret Management

- **Pattern**: Use `stringData` field in Secret manifests for SOPS encryption
- **Encryption**: Run `sops -e -i <file>` after setting values, before committing
- **Verification**: Check `.sops.yaml` file contains `ENC[...]` values in `stringData` section
- **Flux Integration**: Flux automatically decrypts SOPS secrets during reconciliation (requires `sops-age` secret in `flux-system`)

### Home Assistant Configuration Management

- **Pattern**: Use ConfigMap for Recorder configuration, mount via `subPath` to avoid overwriting entire `/config`
- **Environment Variables**: Home Assistant supports `!env_var` directive for referencing environment variables
- **Retry Settings**: `db_max_retries` and `db_retry_wait` help maintain service availability during temporary DB outages
- **Config Reload**: Home Assistant may require restart to pick up ConfigMap changes (not auto-reloaded)

### Migration Job Considerations

- **pgloader Options**: Use `data only`, `drop indexes`, `reset sequences` for cleaner migration
- **Resource Limits**: Migration job needs sufficient CPU/memory (2 CPU, 4Gi memory recommended)
- **PVC Access**: Migration job mounts PVC as read-only to prevent accidental modifications
- **Job Cleanup**: Set `ttlSecondsAfterFinished` to auto-cleanup completed jobs
- **Idempotency**: Migration should be idempotent, but `backoffLimit: 0` prevents automatic retries

### GitOps Workflow

- **Temporary Resources**: Migration Job is added to kustomization temporarily, removed after 24h observation
- **Commit Strategy**: Commit Secret template first, then encrypt, then commit encrypted version
- **Rollback**: Git revert is the primary rollback mechanism (Flux reconciles automatically)

## Gotchas

### Secret Encryption Timing

- **Issue**: Secret must be encrypted with SOPS before committing to Git
- **Solution**: Always run `sops -e -i` after editing Secret files
- **Verification**: Check file contains `ENC[...]` patterns, not plaintext passwords

### ConfigMap Mount Path

- **Issue**: Mounting ConfigMap to `/config/configuration.yaml` requires `subPath` to avoid overwriting entire directory
- **Solution**: Use `subPath: configuration.yaml` in volumeMount
- **Note**: Home Assistant reads configuration from `/config/configuration.yaml` by default

### PostgreSQL Connection String Format

- **Issue**: Connection string must include database name in path
- **Format**: `postgresql://user:password@host:port/database`
- **Example**: `postgresql://homeassistant:pass@postgres-cluster-rw.postgres.svc.cluster.local:5432/homeassistant`

### Migration Job Dependencies

- **Issue**: Migration job needs access to both SQLite file (PVC) and PostgreSQL (Secret)
- **Solution**: Mount PVC as read-only volume, inject DB_URL from Secret as environment variable
- **Verification**: Test job can access both resources before cutover

## Best Practices

### Pre-Cutover Checklist

1. Verify all manifests are committed and Flux-reconciled
2. Test Secret decryption: `sops -d k8s/apps/home-assistant/app/secret-db-url.sops.yaml`
3. Verify PostgreSQL DB/user exists and credentials work
4. Create backups (Longhorn snapshot + CloudNativePG backup)
5. Document backup names/timestamps

### During Cutover

1. Stop Home Assistant first (prevent writes)
2. Monitor migration job logs in real-time
3. Verify job completion before starting Home Assistant
4. Check Home Assistant logs immediately after startup
5. Document actual downtime duration

### Post-Cutover

1. Verify history continuity in UI within first hour
2. Monitor logs for 24 hours for DB connectivity issues
3. Keep SQLite database accessible for rollback window
4. Document any issues or anomalies
5. Remove migration Job from Git after 24h observation

## Troubleshooting Quick Reference

| Issue | Check | Solution |
|-------|-------|----------|
| Secret not decrypting | Flux logs, sops-age secret | Verify SOPS key is correct |
| Home Assistant won't start | Pod logs, Secret/ConfigMap | Check env var injection, ConfigMap mount |
| Migration job fails | Job logs, PVC access | Verify SQLite file exists, PostgreSQL connectivity |
| History missing | PostgreSQL queries, HA logs | Verify migration completed, check Recorder config |
| Rollback needed | Git history, SQLite file | Revert commits, restore from snapshot if needed |

## Future Improvements

- Consider automated backup verification (periodic restore tests)
- Add monitoring/alerting for Recorder DB connectivity
- Document performance characteristics (query times, storage growth)
- Consider connection pooling if multiple Home Assistant instances

## Migration Completion Record

**Date**: 2026-01-10
**Status**: Complete

### Completion Summary

- All tasks completed successfully
- Migration Job executed and completed
- Home Assistant successfully migrated to PostgreSQL
- Migration Job removed from kustomization.yaml after successful migration
- All user stories (US1, US2, US3) validated

### Key Issues Resolved

1. **pgloader Script Dollar Sign Parsing**: Initial attempts to use `$$` in heredoc failed due to YAML parsing issues. Resolved by using placeholder (`DOLLAR_DOLLAR_PLACEHOLDER`) and replacing with sed command.

2. **Job Template Immutability**: Kubernetes Job `spec.template` is immutable. When updating Job manifests, existing Jobs must be deleted before Flux can apply new versions.

### Final State

- Home Assistant Deployment: Running and healthy
- Database: PostgreSQL cluster `postgres-cluster`
- Migration Job: Removed from GitOps (kustomization.yaml)
- Rollback Window: SQLite database retained in PVC for 24 hours

## Notes

- All operational steps should be documented with timestamps
- Keep backup artifacts for at least retention policy duration
- Review and update this document as operational experience accumulates
