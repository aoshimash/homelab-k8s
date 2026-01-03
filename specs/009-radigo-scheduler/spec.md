# Feature Specification: Radigo Scheduled Recorder

**Feature Branch**: `009-radigo-scheduler`  
**Created**: 2026-01-04  
**Status**: Draft  
**Input**: User description: "radigoを定期的に実行してaudiobookshelfのPVCに保存し、audiobookshelfのAPIを実行してRSSを更新する"

## Clarifications

### Session 2026-01-04

- Q: radiko premium認証情報の保存方法は？ → A: SOPS暗号化されたKubernetes Secret（既存パターンに準拠）。ただし現時点ではradiko premiumは使用しない（エリア内放送のみ）
- Q: 古いエピソードの自動削除は？ → A: 自動削除しない（手動管理、ストレージ監視のみ）
- Q: 同時録音の最大数は？ → A: 5番組
- Q: Audiobookshelf APIキーの保存方法は？ → A: SOPS暗号化されたKubernetes Secret（既存パターンに準拠）
- Q: 録音方式の優先度は？ → A: タイムフリー優先。番組の曜日・開始時刻を設定し、radiko番組表APIから終了時刻を自動取得。終了時刻になったらタイムフリー録音を開始する

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Automatic radiko program recording (Priority: P1)

As a podcast listener, I want radiko programs to be automatically recorded on a schedule so that I can listen to my favorite radio programs as podcasts at my convenience.

**Why this priority**: This is the core functionality - without automated recording, the entire feature has no value. Users need their radio programs captured reliably.

**Independent Test**: After configuring a recording schedule, the specified radiko program is recorded and saved as an audio file in the designated storage location.

**Acceptance Scenarios**:

1. **Given** a recording schedule is configured with station ID, day of week, and start time, **When** the system fetches the program info from radiko API and the end time arrives, **Then** radigo begins timefree recording of the program.
2. **Given** radigo is recording a program, **When** the recording completes, **Then** an audio file is saved to the audiobookshelf podcasts storage.
3. **Given** a recording has completed, **When** checking the storage location, **Then** the audio file is named with identifiable information (station ID, date/time, program name).

---

### User Story 2 - Audiobookshelf library integration (Priority: P1)

As a podcast listener, I want recorded programs to appear in my Audiobookshelf podcast library so that I can browse and play them alongside my other podcasts.

**Why this priority**: The recording is useless if users cannot access it through their media interface. Integration with Audiobookshelf makes the recordings discoverable and playable.

**Independent Test**: After a recording completes and the library is refreshed, the new episode appears in Audiobookshelf and can be played.

**Acceptance Scenarios**:

1. **Given** a recording has been saved to the podcasts directory, **When** Audiobookshelf scans the library, **Then** the new episode appears in the appropriate podcast series.
2. **Given** a new episode appears in Audiobookshelf, **When** selecting the episode, **Then** the episode can be streamed with correct metadata (title, duration, date).
3. **Given** multiple recordings from the same radio program exist, **When** viewing the podcast in Audiobookshelf, **Then** all episodes are organized chronologically under the same series.

---

### User Story 3 - Automatic RSS feed update (Priority: P1)

As a podcast listener, I want the RSS feed to be automatically updated after each recording so that I can subscribe to my recorded programs in any podcast app.

**Why this priority**: RSS update enables listening from any podcast client (phone apps, car systems, etc.), not just the Audiobookshelf web interface. This is essential for mobile listening.

**Independent Test**: After a recording completes and RSS is updated, the new episode is available when fetching the RSS feed URL.

**Acceptance Scenarios**:

1. **Given** a recording has been saved, **When** the post-recording process runs, **Then** Audiobookshelf's library scan API is triggered.
2. **Given** the library scan completes, **When** fetching the podcast's RSS feed URL, **Then** the new episode is included in the feed with correct metadata.
3. **Given** a podcast client is subscribed to the RSS feed, **When** the client checks for updates, **Then** the new episode is available for download or streaming.

---

### User Story 4 - Multiple program scheduling with dedicated directories (Priority: P2)

As a podcast listener, I want to configure multiple radio programs to be recorded into separate directories (e.g., `/podcasts/arco`, `/podcasts/ijuin`) so that each program appears as its own podcast series in Audiobookshelf.

**Why this priority**: Most users listen to multiple programs. Organizing each program in its own directory ensures clean separation in the podcast library and proper RSS feed generation per series.

**Independent Test**: With multiple recording schedules configured, each program is recorded to its designated directory and appears as a separate podcast series.

**Acceptance Scenarios**:

1. **Given** multiple recording schedules are configured with distinct output directories, **When** each scheduled time arrives, **Then** the corresponding program is recorded to its designated directory (e.g., `/podcasts/arco/`, `/podcasts/ijuin/`).
2. **Given** recordings exist in `/podcasts/arco/` and `/podcasts/ijuin/`, **When** Audiobookshelf scans the library, **Then** each directory appears as a separate podcast series with its own episodes.
3. **Given** two programs have overlapping broadcast times on different stations, **When** both scheduled times arrive, **Then** both programs are recorded simultaneously to their respective directories.
4. **Given** a schedule configuration exists, **When** viewing the configuration, **Then** all scheduled programs with their station IDs, times, output directories, and frequencies are visible.

---

### User Story 5 - Program title validation before recording (Priority: P2)

As a podcast listener, I want the system to verify the program title before recording so that recordings are skipped when the regular program is pre-empted by a special broadcast or the show is on hiatus.

**Why this priority**: Radio programs occasionally go on hiatus or are replaced by special broadcasts. Recording unrelated content wastes storage and pollutes the podcast library with irrelevant episodes.

**Independent Test**: When the scheduled time slot contains a different program (e.g., special broadcast), the recording is skipped and logged.

**Acceptance Scenarios**:

1. **Given** a recording schedule is configured with an expected program title pattern (e.g., "オードリーのオールナイトニッポン"), **When** the scheduled time arrives and the actual program title matches the pattern, **Then** the recording proceeds normally.
2. **Given** a recording schedule is configured with an expected program title pattern, **When** the scheduled time arrives and the actual program title does NOT match (e.g., "特別番組"), **Then** the recording is skipped.
3. **Given** a recording is skipped due to title mismatch, **When** checking system logs, **Then** the skip is logged with the expected pattern and actual program title for debugging.
4. **Given** a recording schedule with title validation, **When** the configuration is viewed, **Then** the expected title pattern is visible alongside other schedule parameters.

---

### User Story 6 - Recording failure notification and retry (Priority: P3)

As a cluster operator, I want to be notified when a recording fails and have automatic retry logic so that I don't miss programs due to transient errors.

**Why this priority**: Reliability is important but the core functionality must work first. Retry and notification improve operational experience.

**Independent Test**: When a recording fails due to a transient error, the system retries and logs the failure for visibility.

**Acceptance Scenarios**:

1. **Given** a recording attempt fails, **When** the failure is transient (network error), **Then** the system retries the recording within a configured window.
2. **Given** a recording has failed, **When** checking system logs or metrics, **Then** the failure is recorded with error details.
3. **Given** all retry attempts have failed, **When** checking the system state, **Then** the failed recording is logged for manual investigation.

---

### Edge Cases

- What happens when radiko authentication fails (for premium/area-free features)?
- How does the system behave when the recording storage is full? → **Recording fails; no automatic deletion - manual cleanup required**
- What happens when Audiobookshelf is unavailable during RSS update?
- How does the system handle radiko service outages during scheduled recording times?
- What happens when the scheduled program is pre-empted or rescheduled by the broadcaster? → **Recording is skipped via title validation (User Story 5)**
- How does the system behave when ffmpeg (required by radigo) is unavailable or fails?
- What happens when multiple recordings are scheduled at the same time?
- What happens when the radiko program schedule API is unavailable? → **Recording is skipped and logged; retry on next scheduled occurrence**
- How does the system handle partial title matches (e.g., "オードリーのオールナイトニッポン 特別編")?
- What happens when a program directory does not exist yet for the first recording?

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: The system MUST execute radigo on a configurable schedule to record specified radiko programs.
- **FR-002**: The system MUST save recorded audio files to the audiobookshelf podcasts PVC (`audiobookshelf-podcasts`).
- **FR-003**: The system MUST organize recorded files into program-specific subdirectories (e.g., `/podcasts/arco/`, `/podcasts/ijuin/`) where each directory represents a separate podcast series.
- **FR-004**: The system MUST trigger Audiobookshelf's library scan API after each successful recording to update the catalog. API key stored in SOPS-encrypted Kubernetes Secret.
- **FR-005**: The system MUST use radiko timefree as the primary recording method. The system fetches the program's end time from radiko's program schedule API using station ID and start time, then triggers recording at the end time.
- **FR-006**: The system MAY support live stream capture for programs outside the timefree window in future iterations (not in initial scope).
- **FR-007**: The system MUST support configuration of multiple recording schedules for different programs/stations, each with its own output directory.
- **FR-008**: The system MUST output recordings in AAC format (radigo's native output format) to avoid transcoding overhead and quality loss.
- **FR-009**: The system MUST name output files with identifiable information (station ID, timestamp, program title if available).
- **FR-010**: The system SHOULD support radiko premium authentication for area-free recording in the future; credentials will be stored in a SOPS-encrypted Kubernetes Secret following existing cluster patterns. (Not used in initial deployment - local area broadcasts only)
- **FR-011**: The system SHOULD implement retry logic for transient failures during recording.
- **FR-012**: The system SHOULD expose metrics or logs for monitoring recording success/failure.
- **FR-013**: The system MUST be deployed via GitOps (Flux) following the existing cluster patterns.
- **FR-014**: The system MUST validate the program title before recording by checking if the actual program title contains the configured expected title pattern.
- **FR-015**: The system MUST skip recording and log the event when the program title validation fails (e.g., program on hiatus, replaced by special broadcast).
- **FR-016**: The system MUST automatically create the output directory if it does not exist for the first recording of a program.
- **FR-017**: Each recording schedule MUST include a configurable expected title pattern for validation (supports partial/substring matching).
- **FR-018**: The system MUST fetch program end time and official title from radiko's program schedule API (`/v3/program/station/weekly/{station_id}.xml`) before each recording.

### Key Entities

- **Recording Schedule**: Configuration defining when and what to record, including:
  - Station ID (e.g., `LFR`, `TBS`, `QRR`)
  - Day of week
  - Start time (broadcast start; end time is auto-fetched from radiko program API)
  - Output directory (e.g., `/podcasts/arco/`)
  - Expected title pattern for validation (e.g., "オードリーのオールナイトニッポン")
- **Recorded Episode**: Audio file output from radigo with associated metadata (filename, duration, recording date).
- **Podcast Series**: Collection of episodes from the same radio program, stored in a dedicated directory (e.g., `/podcasts/arco/`) and appearing as a separate podcast in Audiobookshelf.
- **Program Info**: Metadata retrieved from radiko about the current/scheduled program, used for title validation before recording.
- **Audiobookshelf API**: Interface to trigger library scans and RSS feed updates after recordings complete.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: A configured program is successfully recorded within 5 minutes of its scheduled time.
- **SC-002**: Recorded episodes appear in Audiobookshelf library within 10 minutes of recording completion.
- **SC-003**: The podcast RSS feed includes new episodes within 15 minutes of recording completion.
- **SC-004**: 95% of scheduled recordings complete successfully over a 30-day period under normal conditions.
- **SC-005**: Users can stream recorded episodes from Audiobookshelf with playback starting within 5 seconds.
- **SC-006**: Up to 5 simultaneous recordings complete without resource conflicts.
- **SC-007**: Each program's recordings appear as a separate podcast series in Audiobookshelf (one series per output directory).
- **SC-008**: 100% of recordings with mismatched program titles are correctly skipped (no false recordings).
- **SC-009**: Skipped recordings due to title mismatch are logged with sufficient detail for debugging (expected pattern, actual title, timestamp).

## Assumptions

- The cluster has the audiobookshelf deployment running with the `audiobookshelf-podcasts` PVC (spec 005).
- Audiobookshelf provides an API endpoint for triggering library scans (to be verified during implementation).
- Radigo container image is available from a container registry or will be built as part of this feature.
- Network access to radiko.jp is available from the cluster for recording streams.
- The PVC `audiobookshelf-podcasts` has `ReadWriteOnce` access mode; the radigo scheduler must run on the same node or use a shared storage solution.
- Radiko timefree allows recording of programs broadcast within the last week.
- AAC format is natively supported by Audiobookshelf for playback and RSS feed generation (no transcoding required).
