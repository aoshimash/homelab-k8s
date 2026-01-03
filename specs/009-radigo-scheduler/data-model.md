# Data Model: Radigo Scheduled Recorder

## Entities

### 1. Recording Schedule (ConfigMap)

Configuration for each program to record.

```yaml
# ConfigMap: radigo-schedules
data:
  schedules.json: |
    {
      "programs": [
        {
          "name": "arco",
          "station_id": "LFR",
          "day_of_week": 6,           # 0=Sun, 6=Sat
          "start_time": "01:00",      # JST (25:00 in radiko notation)
          "expected_title": "オードリーのオールナイトニッポン",
          "output_dir": "/podcasts/arco"
        },
        {
          "name": "ijuin",
          "station_id": "TBS",
          "day_of_week": 1,           # Monday
          "start_time": "01:00",      # JST (25:00)
          "expected_title": "伊集院光 深夜の馬鹿力",
          "output_dir": "/podcasts/ijuin"
        }
      ]
    }
```

| Field | Type | Description | Validation |
|-------|------|-------------|------------|
| name | string | Program identifier (used in CronJob name) | lowercase, alphanumeric |
| station_id | string | Radiko station ID | Must be valid radiko station |
| day_of_week | integer | 0-6 (Sunday-Saturday) | 0-6 |
| start_time | string | Broadcast start time (HH:MM JST) | Valid time format |
| expected_title | string | Title pattern for validation | Non-empty string |
| output_dir | string | Output directory path | Absolute path starting with /podcasts/ |

### 2. Audiobookshelf Credentials (Secret)

```yaml
# Secret: radigo-audiobookshelf-api (SOPS encrypted)
data:
  api_key: <base64-encoded-api-key>
  library_id: <base64-encoded-library-id>
  base_url: <base64-encoded-url>  # e.g., http://audiobookshelf.audiobookshelf.svc.cluster.local
```

| Field | Type | Description |
|-------|------|-------------|
| api_key | string | Audiobookshelf API key |
| library_id | string | Podcast library UUID |
| base_url | string | Audiobookshelf internal service URL |

### 3. Recorded Episode (File)

Output file from radigo recording.

| Attribute | Format | Example |
|-----------|--------|---------|
| Filename | `{YYYYMMDD}-{station_id}.aac` | `20260104-LFR.aac` |
| Location | `{output_dir}/{filename}` | `/podcasts/arco/20260104-LFR.aac` |
| Format | AAC | radigo native output |

### 4. Program Info (API Response - ephemeral)

Data fetched from radiko API before each recording.

```xml
<!-- From /v3/program/station/weekly/{station_id}.xml -->
<prog ft="20260104010000" to="20260104030000" dur="7200">
  <title>オードリーのオールナイトニッポン</title>
</prog>
```

| Field | Type | Description |
|-------|------|-------------|
| ft | string | From time (YYYYMMDDHHmmss) |
| to | string | To time (YYYYMMDDHHmmss) |
| title | string | Official program title |

## State Transitions

### Recording Job Lifecycle

```
[Scheduled] → [Running] → [Completed/Failed]
     │            │
     │            ├─→ [Title Mismatch - Skipped]
     │            │
     │            ├─→ [API Error - Retry]
     │            │
     │            └─→ [Recording Error - Retry]
     │
     └─→ (waits for next cron trigger)
```

### Recording Flow States

1. **INIT**: CronJob triggered, pod starting
2. **FETCHING_SCHEDULE**: Querying radiko API for program info
3. **VALIDATING**: Checking title against expected pattern
4. **SKIPPED**: Title mismatch, job exits successfully (not an error)
5. **RECORDING**: radigo `rec` command executing
6. **MOVING**: Renaming/moving output file to target directory
7. **SCANNING**: Triggering Audiobookshelf library scan
8. **COMPLETED**: All steps successful
9. **FAILED**: Error occurred, may retry based on backoffLimit

## Relationships

```
┌─────────────────────┐
│  Recording Schedule │
│    (ConfigMap)      │
└─────────┬───────────┘
          │ 1:N
          ▼
┌─────────────────────┐
│      CronJob        │
│  (per program)      │
└─────────┬───────────┘
          │ triggers
          ▼
┌─────────────────────┐      ┌─────────────────────┐
│    Recording Job    │─────▶│   Recorded Episode  │
│      (Pod)          │      │       (File)        │
└─────────┬───────────┘      └─────────────────────┘
          │ calls
          ▼
┌─────────────────────┐      ┌─────────────────────┐
│   Radiko API        │      │  Audiobookshelf API │
│ (program schedule)  │      │   (library scan)    │
└─────────────────────┘      └─────────────────────┘
```

## Volume Mounts

| Mount Point | PVC/Source | Access | Purpose |
|-------------|------------|--------|---------|
| /podcasts | audiobookshelf-podcasts | ReadWriteOnce | Recording output |
| /output | emptyDir | ReadWrite | radigo working directory |
| /scripts | ConfigMap | ReadOnly | Recording script |
