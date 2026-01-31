# Research: Migrate radigo to yt-dlp-rajiko

**Feature**: 015-migrate-radigo-yt-dlp  
**Date**: 2026-01-31

## Research Summary

All technical questions have been resolved through documentation review and existing codebase analysis.

---

## 1. yt-dlp-rajiko Installation Method

**Decision**: Use pip installation in Docker image

**Rationale**:
- pip provides clean dependency management
- yt-dlp-rajiko v1.11 is the latest stable version
- Requires yt-dlp 2025.02.19 or newer

**Alternatives Considered**:
- Bundle zip file: More complex, manual version management
- pipx: Overkill for containerized environment

**Implementation**:
```dockerfile
RUN pip install --no-cache-dir yt-dlp yt-dlp-rajiko
```

---

## 2. yt-dlp CLI Usage for Timefree Recording

**Decision**: Use radiko timefree URL format with yt-dlp

**Rationale**:
- yt-dlp-rajiko supports timefree URLs directly
- Area-free works without authentication for 7-day timefree
- Metadata embedding is built-in with `--embed-metadata --embed-thumbnail`

**URL Format**:
```
https://radiko.jp/#!/ts/{STATION_ID}/{YYYYMMDDHHMMSS}
```

**Example Command**:
```bash
yt-dlp --embed-metadata --embed-thumbnail -N 10 \
  -o "/output/%(title)s %(timestamp>%Y-%m-%d)s.%(ext)s" \
  "https://radiko.jp/#!/ts/TBS/20260131010000"
```

**Alternatives Considered**:
- radiko share URL format: Works but timefree URL is more explicit
- Live recording: Not needed per spec (timefree only)

---

## 3. Output File Format

**Decision**: Use yt-dlp default format (M4A with AAC audio)

**Rationale**:
- yt-dlp-rajiko outputs M4A by default (same as radiko stream format)
- Compatible with audiobookshelf podcast library
- Metadata embedding works natively with M4A container

**Filename Template** (using yt-dlp default with customization):
```
%(title)s %(timestamp>%Y-%m-%d)s [%(id)s].%(ext)s
```

**Alternatives Considered**:
- MP3 conversion: Additional processing, quality loss
- Keep radigo format: Different tool, different conventions

---

## 4. Error Handling and Retry Strategy

**Decision**: Kubernetes Job backoffLimit=3

**Rationale**:
- Matches clarified requirement from spec
- Simple, proven approach for transient failures
- No complex script-level retry logic needed

**Configuration**:
```yaml
spec:
  backoffLimit: 3
  ttlSecondsAfterFinished: 86400
```

**Alternatives Considered**:
- Script-level exponential backoff: More complex, harder to debug
- No retry: Risky for transient network issues

---

## 5. Docker Base Image Selection

**Decision**: python:3.12-slim

**Rationale**:
- yt-dlp and yt-dlp-rajiko are Python packages
- slim variant reduces image size
- ffmpeg available via apt for audio processing

**Alternatives Considered**:
- ubuntu:24.04: Larger, more dependencies to manage manually
- alpine: pip installation can be problematic with some packages

---

## 6. Migration Strategy

**Decision**: Simultaneous replacement (per clarification)

**Rationale**:
- User chose immediate replacement over gradual migration
- Simplifies deployment - single atomic change
- Old radigo image deleted in same commit

**Implementation Steps**:
1. Create new yt-dlp-rajiko Docker image
2. Update configmap-record-script.yaml with new script
3. Update cronjob.yaml to use new image
4. Delete images/radigo/ directory
5. Commit and push - Flux reconciles all changes

**Alternatives Considered**:
- Parallel deployment: More complex, user rejected
- Gradual program-by-program migration: Unnecessary complexity

---

## 7. Radiko URL Construction

**Decision**: Construct URL from existing environment variables

**Rationale**:
- Existing CronJob passes STATION_ID, DAY_OF_WEEK, START_TIME
- Can reuse date calculation logic from existing script
- URL format: `https://radiko.jp/#!/ts/{STATION_ID}/{YYYYMMDDHHMMSS}`

**Mapping**:
| Existing Env Var | Usage |
|------------------|-------|
| STATION_ID | Direct use in URL |
| DAY_OF_WEEK | Calculate target date |
| START_TIME | Convert to HHMMSS format |
| EXPECTED_TITLE | Title validation (optional) |
| OUTPUT_DIR | Final file destination |

---

## 8. Program Validation

**Decision**: Simplify validation using yt-dlp output

**Rationale**:
- yt-dlp-rajiko extracts title from radiko API automatically
- Can validate after download by checking file metadata
- Reduces script complexity vs current XML parsing

**Alternatives Considered**:
- Keep full XML validation: Complex, already handled by yt-dlp
- Skip validation entirely: Risky, could download wrong program

---

## Dependencies Summary

| Component | Version | Source |
|-----------|---------|--------|
| yt-dlp | ≥2025.02.19 | pip |
| yt-dlp-rajiko | v1.11 | pip |
| ffmpeg | latest | apt |
| Python | 3.12 | base image |
