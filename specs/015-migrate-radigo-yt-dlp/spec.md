# Feature Specification: Migrate radigo recorder to yt-dlp-rajiko

**Feature Branch**: `015-migrate-radigo-yt-dlp`  
**Created**: 2026-01-31  
**Status**: Implementation Complete  
**Input**: User description: "issue#126に対応 - Migrate radigo recorder to yt-dlp-rajiko due to radiko API changes"  
**Related Issue**: [#126](https://github.com/aoshimash/homelab-k8s/issues/126)

## Clarifications

### Session 2026-01-31

- Q: 録音失敗時のリトライ動作はどうすべきか？ → A: Kubernetesジョブレベルで固定回数リトライ（3回）
- Q: 古いradigoイメージの削除タイミングは？ → A: 新イメージデプロイと同時に即削除
- Q: 録音ファイルの命名規則は？ → A: yt-dlpのデフォルト形式を使用
- Q: ストレージ容量不足時の動作は？ → A: 録音を試行し、失敗時にエラーログを出力（Kubernetes標準動作）

## Background

The current radigo-based radiko recorder is failing due to radiko API changes (January 28, 2026). The radigo tool (v0.12.0, last updated October 2022) is no longer maintained and cannot handle the new playlist URL structure (`https://tf-*-rpaa-radiko.smartstream.ne.jp`).

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Record Scheduled Radio Programs (Priority: P1)

As a homelab user, I want my scheduled radio programs (ijuin, audrey, arco, ariyoshi) to be automatically recorded so that I can listen to them at my convenience.

**Why this priority**: This is the core functionality that is currently broken. Without working recordings, the entire radiko recording system is unusable.

**Independent Test**: Can be tested by triggering a recording job for any program and verifying that the audio file is successfully downloaded and stored.

**Acceptance Scenarios**:

1. **Given** a scheduled recording CronJob is configured, **When** the scheduled time arrives, **Then** the recording job starts and successfully downloads the audio content
2. **Given** a recording job is running, **When** the download completes, **Then** the audio file is saved in the correct format and location
3. **Given** a recording job is running, **When** the radiko stream is temporarily unavailable, **Then** the Kubernetes Job retries up to 3 times and logs each attempt

---

### User Story 2 - Maintain Recording Quality and Metadata (Priority: P2)

As a homelab user, I want my recorded radio programs to have proper audio quality and embedded metadata so that I can organize and identify my recordings easily.

**Why this priority**: Quality and metadata enhance user experience but are secondary to the basic recording functionality working.

**Independent Test**: Can be tested by recording a program and inspecting the output file for audio quality and embedded metadata (title, artist, cover art).

**Acceptance Scenarios**:

1. **Given** a recording completes successfully, **When** I check the audio file, **Then** it contains proper audio quality comparable to the original stream
2. **Given** a recording completes successfully, **When** I inspect the file metadata, **Then** it includes relevant information (program name, date, station)

---

### User Story 3 - Monitor Recording Status (Priority: P3)

As a homelab administrator, I want to monitor the recording jobs' success and failure status so that I can troubleshoot issues when recordings fail.

**Why this priority**: Monitoring is important for maintenance but can be addressed after core functionality works.

**Independent Test**: Can be tested by checking Kubernetes job logs and verifying error messages are clear and actionable.

**Acceptance Scenarios**:

1. **Given** a recording job fails, **When** I check the job logs, **Then** I can see clear error messages indicating the cause of failure
2. **Given** recording jobs are running, **When** I query Kubernetes, **Then** I can see the job status and history

---

### Edge Cases

- What happens when radiko service is down during scheduled recording time?
  - → Job retries up to 3 times, then fails with error log
- How does the system handle network interruptions during download?
  - → Job retries up to 3 times at Kubernetes level
- What happens when the storage volume is full?
  - → Recording attempt proceeds and fails with error log (standard Kubernetes behavior)
- How does the system handle concurrent recording of multiple programs?
  - → Each program runs as independent CronJob; no coordination needed
- What happens when the yt-dlp-rajiko plugin is outdated and needs updating?
  - → Requires manual Docker image rebuild and redeployment

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: System MUST successfully download radiko time-free (recorded) radio programs using yt-dlp with yt-dlp-rajiko plugin
- **FR-002**: System MUST support all currently configured programs (ijuin, audrey, arco, ariyoshi)
- **FR-003**: System MUST save recordings in a playable audio format (AAC/M4A or equivalent) using yt-dlp default filename format
- **FR-004**: System MUST run recordings on scheduled times via Kubernetes CronJobs
- **FR-005**: System MUST handle recording failures gracefully with appropriate logging and retry up to 3 times at Kubernetes Job level
- **FR-006**: System MUST embed available metadata (program name, station, date) in the audio file
- **FR-007**: System MUST store recordings in the configured persistent storage location
- **FR-008**: System MUST provide clear error messages when recordings fail

### Key Entities

- **Recording Job**: A Kubernetes CronJob that triggers at scheduled times to record specific radio programs
- **Recording Configuration**: ConfigMap containing the recording script and program parameters
- **Container Image**: Docker image containing yt-dlp and yt-dlp-rajiko plugin
- **Audio File**: The output recording saved to persistent storage

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: All four configured programs (ijuin, audrey, arco, ariyoshi) can be recorded successfully
- **SC-002**: Recording should succeed reliably under normal conditions (target: 95%+ when manually audited)
- **SC-003**: Audio files are playable and contain complete program content
- **SC-004**: Recording failures produce actionable error messages within 30 seconds of failure
- **SC-005**: Migration can be completed with zero data loss for existing scheduled recordings
- **SC-006**: Old radigo Docker image is removed simultaneously with new image deployment

## Assumptions

- radiko time-free service remains available and accessible
- yt-dlp-rajiko plugin continues to be maintained and compatible with radiko API changes
- Existing storage configuration and volume mounts remain unchanged
- Kubernetes cluster infrastructure remains stable during migration
- No premium radiko account is required (using area-free capability of yt-dlp-rajiko)

## Out of Scope

- Live streaming (real-time) recording capability
- Changes to storage infrastructure or backup systems
- User interface for managing recordings
- Automatic retry scheduling for failed recordings
- Integration with podcast/media servers
