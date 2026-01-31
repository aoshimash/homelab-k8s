# Tasks: Migrate radigo to yt-dlp-rajiko

**Input**: Design documents from `/specs/015-migrate-radigo-yt-dlp/`  
**Prerequisites**: plan.md ✓, spec.md ✓, research.md ✓, data-model.md ✓, quickstart.md ✓

**Tests**: Manual verification via kubectl (per spec) - no automated test tasks included

**Organization**: Tasks are grouped by user story to enable independent implementation and testing of each story.

## Format: `[ID] [P?] [Story] Description`

- **[P]**: Can run in parallel (different files, no dependencies)
- **[Story]**: Which user story this task belongs to (e.g., US1, US2, US3)
- Include exact file paths in descriptions

## Path Conventions

- **Docker images**: `images/yt-dlp-rajiko/`
- **Kubernetes manifests**: `k8s/apps/audiobookshelf/radigo-recorder/`
- **Old image (to delete)**: `images/radigo/`

---

## Phase 1: Setup (Docker Image)

**Purpose**: Create new yt-dlp-rajiko container image

- [x] T001 Create directory structure for new image at `images/yt-dlp-rajiko/`
- [x] T002 Create Dockerfile at `images/yt-dlp-rajiko/Dockerfile` with python:3.12-slim base, ffmpeg, yt-dlp, and yt-dlp-rajiko

**Checkpoint**: Docker image definition ready for build

---

## Phase 2: Foundational (Recording Script)

**Purpose**: Core recording script that MUST be complete before user story validation

**⚠️ CRITICAL**: Recording script is the foundation for all user stories

- [x] T003 Rewrite recording script in `k8s/apps/audiobookshelf/radigo-recorder/configmap-record-script.yaml` to use yt-dlp CLI
- [x] T004 Implement date calculation logic for timefree URL construction in record.sh
- [x] T005 Add yt-dlp command with --embed-metadata --embed-thumbnail -N 10 options in record.sh

**Checkpoint**: Foundation ready - recording script complete with yt-dlp integration

---

## Phase 3: User Story 1 - Record Scheduled Radio Programs (Priority: P1) 🎯 MVP

**Goal**: Enable successful recording of all four programs (ijuin, audrey, arco, ariyoshi) using yt-dlp-rajiko

**Independent Test**: Trigger a recording job manually with `kubectl create job --from=cronjob/radigo-ijuin test-ijuin -n audiobookshelf` and verify audio file is downloaded

### Implementation for User Story 1

- [x] T006 [P] [US1] Update image reference in `k8s/apps/audiobookshelf/radigo-recorder/base/cronjob.yaml` to `ghcr.io/aoshimash/homelab-k8s/yt-dlp-rajiko:v1.0.0`
- [x] T007 [P] [US1] Update backoffLimit to 3 in `k8s/apps/audiobookshelf/radigo-recorder/base/cronjob.yaml`
- [x] T008 [P] [US1] Remove RADIGO_HOME environment variable from `k8s/apps/audiobookshelf/radigo-recorder/base/cronjob.yaml`
- [x] T009 [US1] Delete old radigo image directory `images/radigo/`

**Checkpoint**: User Story 1 complete - recording jobs can successfully download audio files

---

## Phase 4: User Story 2 - Maintain Recording Quality and Metadata (Priority: P2)

**Goal**: Ensure recorded files have proper audio quality and embedded metadata (title, artist, cover art)

**Independent Test**: Record a program and inspect with `ffprobe` to verify metadata fields are populated

### Implementation for User Story 2

- [x] T010 [US2] Verify yt-dlp metadata flags (--embed-metadata, --embed-thumbnail) are correctly set in record.sh
- [x] T011 [US2] Add output filename template using yt-dlp default format in record.sh

**Checkpoint**: User Story 2 complete - audio files contain embedded metadata

---

## Phase 5: User Story 3 - Monitor Recording Status (Priority: P3)

**Goal**: Enable clear monitoring of recording job success/failure with actionable error messages

**Independent Test**: Trigger a job failure (e.g., invalid station ID) and verify error logs are clear and actionable

### Implementation for User Story 3

- [x] T012 [US3] Add structured logging with timestamps to record.sh
- [x] T013 [US3] Add error message output for common failure scenarios (network, auth, storage) in record.sh
- [x] T014 [US3] Ensure non-zero exit code on failure for Kubernetes Job status tracking in record.sh

**Checkpoint**: User Story 3 complete - job logs provide actionable error information

---

## Phase 6: Polish & Deployment

**Purpose**: Final cleanup and deployment preparation

- [x] T015 [P] Build and push Docker image to ghcr.io/aoshimash/homelab-k8s/yt-dlp-rajiko:v1.0.0
- [x] T016 Verify GitHub Actions workflow handles new image (check `.github/workflows/`)
- [x] T017 Run manual test recording for one program (ijuin) to validate end-to-end
- [ ] T018 Run quickstart.md validation steps (optional, post-merge)
- [x] T019 Update spec.md status from Draft to Complete

---

## Dependencies & Execution Order

### Phase Dependencies

- **Setup (Phase 1)**: No dependencies - can start immediately
- **Foundational (Phase 2)**: Depends on Setup completion - BLOCKS all user stories
- **User Story 1 (Phase 3)**: Depends on Foundational phase completion
- **User Story 2 (Phase 4)**: Can start after Foundational (mostly verification of T005)
- **User Story 3 (Phase 5)**: Can start after Foundational
- **Polish (Phase 6)**: Depends on all user stories being complete

### User Story Dependencies

- **User Story 1 (P1)**: Requires Phase 1 + Phase 2 - Core functionality
- **User Story 2 (P2)**: Requires Phase 2 - Metadata is handled by yt-dlp flags in T005
- **User Story 3 (P3)**: Requires Phase 2 - Logging enhancements to existing script

### Task Dependencies Within Phases

```
Phase 1: T001 → T002
Phase 2: T003 → T004 → T005
Phase 3: T006, T007, T008 can run in parallel → T009 (deletion last)
Phase 4: T010, T011 (verification tasks, mostly already done)
Phase 5: T012 → T013 → T014
Phase 6: T015 → T016 → T017 → T018 → T019
```

### Parallel Opportunities

- T006, T007, T008 can run in parallel (different sections of same file, but atomic changes)
- T010, T011 are verification/implementation tasks (parallel)
- T012, T013 can be developed together (same file, different functions)

---

## Parallel Example: User Story 1

```bash
# These tasks modify different parts of cronjob.yaml and can be done together:
Task T006: "Update image reference in cronjob.yaml"
Task T007: "Update backoffLimit to 3 in cronjob.yaml"
Task T008: "Remove RADIGO_HOME env var from cronjob.yaml"

# Then delete old image:
Task T009: "Delete images/radigo/ directory"
```

---

## Implementation Strategy

### MVP First (User Story 1 Only)

1. Complete Phase 1: Setup (T001-T002) - Create Docker image definition
2. Complete Phase 2: Foundational (T003-T005) - Rewrite recording script
3. Complete Phase 3: User Story 1 (T006-T009) - Update K8s configs, delete old image
4. **STOP and VALIDATE**: Test recording with `kubectl create job --from=cronjob/radigo-ijuin`
5. Deploy if working

### Incremental Delivery

1. Setup + Foundational → Recording script ready
2. User Story 1 → Basic recording works → **MVP Complete**
3. User Story 2 → Metadata verified (mostly already done via yt-dlp flags)
4. User Story 3 → Enhanced logging for troubleshooting
5. Polish → Image pushed, full validation

### Key Files Modified

| File | Action | Phase |
|------|--------|-------|
| `images/yt-dlp-rajiko/Dockerfile` | CREATE | 1 |
| `k8s/apps/audiobookshelf/radigo-recorder/configmap-record-script.yaml` | UPDATE | 2 |
| `k8s/apps/audiobookshelf/radigo-recorder/base/cronjob.yaml` | UPDATE | 3 |
| `images/radigo/Dockerfile` | DELETE | 3 |

---

## Notes

- [P] tasks = different files, no dependencies
- [Story] label maps task to specific user story for traceability
- Each user story should be independently completable and testable
- Commit after each phase or logical group
- Stop at any checkpoint to validate story independently
- All changes applied via GitOps (Flux reconciliation after push)
