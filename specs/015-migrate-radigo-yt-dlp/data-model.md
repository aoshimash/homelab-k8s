# Data Model: Migrate radigo to yt-dlp-rajiko

**Feature**: 015-migrate-radigo-yt-dlp  
**Date**: 2026-01-31

## Overview

This feature involves infrastructure configuration changes, not traditional data modeling. The "entities" are Kubernetes resources and Docker artifacts.

---

## Kubernetes Resources

### 1. CronJob (Updated)

**Resource**: `k8s/apps/audiobookshelf/radigo-recorder/base/cronjob.yaml`

| Field | Current Value | New Value |
|-------|---------------|-----------|
| `spec.jobTemplate.spec.backoffLimit` | 2 | 3 |
| `containers[0].image` | `ghcr.io/aoshimash/homelab-k8s/radigo:v0.12.0` | `ghcr.io/aoshimash/homelab-k8s/yt-dlp-rajiko:v1.0.0` |
| `containers[0].env.RADIGO_HOME` | `/output` | (removed) |

**State Transitions**:
```
Scheduled → Running → Succeeded/Failed
                ↓
          (retry up to 3x on failure)
```

### 2. ConfigMap (Updated)

**Resource**: `k8s/apps/audiobookshelf/radigo-recorder/configmap-record-script.yaml`

| Field | Description |
|-------|-------------|
| `data.entrypoint.sh` | Entry point script (unchanged structure) |
| `data.record.sh` | Recording script (complete rewrite for yt-dlp) |

**Environment Variables** (unchanged interface):

| Variable | Type | Description | Example |
|----------|------|-------------|---------|
| `STATION_ID` | string | Radiko station identifier | `TBS`, `LFR` |
| `DAY_OF_WEEK` | int | Day of week (0=Sunday) | `0`-`6` |
| `START_TIME` | string | Broadcast start time | `01:00` |
| `EXPECTED_TITLE` | string | Expected program title substring | `深夜の馬鹿力` |
| `OUTPUT_DIR` | string | Output directory path | `/podcasts/ijuin` |

### 3. Docker Image (New)

**Resource**: `images/yt-dlp-rajiko/Dockerfile`

| Layer | Contents |
|-------|----------|
| Base | `python:3.12-slim` |
| System packages | `ffmpeg`, `curl` |
| Python packages | `yt-dlp`, `yt-dlp-rajiko` |
| Working directory | `/output` |

**Build Arguments**:
- None required (uses latest pip packages)

### 4. Docker Image (Deleted)

**Resource**: `images/radigo/Dockerfile`

- Entire directory to be removed
- GitHub Actions workflow update may be needed if image-specific

---

## File Artifacts

### Audio Output File

| Attribute | Value |
|-----------|-------|
| Format | M4A (AAC audio) |
| Naming | yt-dlp default: `{title} {date} [{id}].m4a` |
| Location | `{OUTPUT_DIR}/` |
| Metadata | Embedded (title, description, thumbnail, tracklist) |

**Example Filename**:
```
伊集院光 深夜の馬鹿力 2026-01-28 [20260128010000-TBS].m4a
```

---

## Relationships

```
┌─────────────────┐     triggers     ┌─────────────────┐
│    CronJob      │ ───────────────► │      Job        │
│  (per program)  │                  │   (instance)    │
└─────────────────┘                  └────────┬────────┘
                                              │
                                              │ creates
                                              ▼
┌─────────────────┐     mounts       ┌─────────────────┐
│   ConfigMap     │ ◄─────────────── │      Pod        │
│ (record script) │                  │ (yt-dlp-rajiko) │
└─────────────────┘                  └────────┬────────┘
                                              │
                                              │ writes
                                              ▼
┌─────────────────┐                  ┌─────────────────┐
│      PVC        │ ◄─────────────── │   Audio File    │
│ (podcasts vol)  │     stores       │    (.m4a)       │
└─────────────────┘                  └─────────────────┘
```

---

## Validation Rules

### CronJob Validation
- Schedule must be valid cron expression
- `backoffLimit` must be ≥ 1

### Environment Variable Validation
- `STATION_ID`: Non-empty, valid radiko station code
- `DAY_OF_WEEK`: Integer 0-6
- `START_TIME`: Format `HH:MM`
- `OUTPUT_DIR`: Absolute path starting with `/podcasts/`

### Output Validation
- File exists after job completion
- File size > 0 bytes
- File is playable M4A format
