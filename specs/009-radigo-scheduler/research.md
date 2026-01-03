# Research: Radigo Scheduled Recorder

## 1. Radigo Container Image

**Decision**: Use official Docker Hub image `yyoshiki41/radigo`

**Rationale**: 
- Official image maintained by radigo author
- Includes ffmpeg and all dependencies
- Regularly updated with releases

**Alternatives considered**:
- Build custom image: Rejected (unnecessary maintenance burden)
- Use radigo binary directly: Rejected (requires ffmpeg installation)

**Usage**:
```bash
docker pull yyoshiki41/radigo

# Timefree recording example
docker run -v $(pwd)/output:/output yyoshiki41/radigo rec -id=LFR -s=20260104010000
```

## 2. Audiobookshelf Library Scan API

**Decision**: Use `POST /api/libraries/:id/scan` endpoint

**Rationale**:
- Official documented API endpoint
- Triggers immediate library scan
- Returns 200 OK on success

**API Details**:
```bash
# Trigger library scan
curl -X POST "http://audiobookshelf.audiobookshelf.svc.cluster.local/api/libraries/<library_id>/scan" \
     -H "Authorization: Bearer <api_key>"
```

**Alternatives considered**:
- Automatic folder watching: Audiobookshelf has this but adds latency
- Direct database modification: Rejected (unsupported, risky)

**Implementation Notes**:
- Need to get library ID from Audiobookshelf (one-time setup via UI)
- API key generated in Audiobookshelf Settings → API Keys

## 3. Radiko Program Schedule API

**Decision**: Use `/v3/program/station/weekly/{station_id}.xml` endpoint

**Rationale**:
- Public API, no authentication required
- Returns full week schedule with start/end times
- XML format with consistent structure

**API Details**:
```
GET http://radiko.jp/v3/program/station/weekly/LFR.xml
```

**Response Structure** (relevant fields):
```xml
<prog ft="20260104010000" to="20260104030000" dur="7200">
  <title>オードリーのオールナイトニッポン</title>
  <pfm>オードリー</pfm>
</prog>
```

- `ft`: from time (start) - YYYYMMDDHHmmss format
- `to`: to time (end) - YYYYMMDDHHmmss format
- `dur`: duration in seconds
- `title`: program title for validation

**Alternatives considered**:
- `/v3/program/today/{area}.xml`: Only today's schedule
- `/v3/program/date/{date}/{area}.xml`: Date-specific but area-based

## 4. PVC Sharing Strategy

**Decision**: Schedule radigo CronJob pods on same node as audiobookshelf using node affinity

**Rationale**:
- `audiobookshelf-podcasts` PVC has `ReadWriteOnce` access mode
- Only one node can mount RWO PVC at a time
- Node affinity ensures CronJob pods run on same node

**Implementation**:
```yaml
spec:
  template:
    spec:
      affinity:
        podAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            - labelSelector:
                matchLabels:
                  app: audiobookshelf
              topologyKey: kubernetes.io/hostname
```

**Alternatives considered**:
- Change PVC to ReadWriteMany: Requires Longhorn RWX (NFS), adds complexity
- Use separate PVC with rsync: Adds latency and complexity
- Run radigo as sidecar: Rejected (different lifecycle, always running)

## 5. Recording Script Logic

**Decision**: Shell script orchestrating radigo with pre/post processing

**Script Flow**:
1. Fetch program info from radiko API (validate title, get end time)
2. If title doesn't match expected pattern → skip and log
3. Calculate timefree recording parameters
4. Execute radigo `rec` command
5. Move output to program-specific directory
6. Trigger Audiobookshelf library scan API
7. Log success/failure

**Rationale**:
- Simple, debuggable, no additional runtime dependencies
- ConfigMap for script, easy to update via GitOps

**Alternatives considered**:
- Python script: Overkill for this use case
- Go binary: Would need to build/maintain custom image

## 6. CronJob Scheduling Strategy

**Decision**: One CronJob per program, triggered at broadcast end time

**Rationale**:
- Independent failure isolation per program
- Clear scheduling (each program has its own cron expression)
- Easy to add/remove programs

**Example Schedule** (オードリーのANN - 土曜 25:00-27:00):
```yaml
# Trigger at 03:05 JST Sunday (27:05 Saturday in radiko time)
# 5 minutes after end to ensure timefree availability
schedule: "5 3 * * 0"  # 03:05 every Sunday (JST)
```

**Alternatives considered**:
- Single CronJob with multiple schedules: Complex, single point of failure
- Kubernetes Job with external scheduler: Overkill

## 7. Directory Structure for Podcasts

**Decision**: `/podcasts/{program-name}/` with timestamped files

**Structure**:
```
/podcasts/
├── arco/                    # オードリーのANN
│   ├── 20260104-LFR.aac
│   ├── 20260111-LFR.aac
│   └── ...
├── ijuin/                   # 伊集院光深夜の馬鹿力
│   ├── 20260107-TBS.aac
│   └── ...
└── ...
```

**Rationale**:
- Audiobookshelf recognizes each directory as separate podcast
- Filename includes date for sorting
- Station ID for debugging (multiple stations possible)

**Alternatives considered**:
- Flat structure: Audiobookshelf wouldn't separate into series
- Nested by date: Overcomplicated, not needed

## 8. Error Handling and Retry

**Decision**: CronJob `restartPolicy: OnFailure` with `backoffLimit: 2`

**Rationale**:
- Kubernetes handles basic retry
- Timefree window (1 week) allows manual retry if all attempts fail
- Logs available via `kubectl logs`

**Alternatives considered**:
- Custom retry logic in script: Adds complexity
- No retry: Risk of missing recordings due to transient errors
