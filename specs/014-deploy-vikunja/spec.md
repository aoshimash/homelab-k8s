# Feature Specification: Deploy Vikunja

**Feature Branch**: `014-deploy-vikunja`  
**Created**: 2026-01-10  
**Status**: Draft  
**Input**: User description: "vikunjaをデプロイ https://github.com/aoshimash/homelab-k8s/issues/88"

## Clarifications

### Session 2026-01-10

- Q: User creation method → A: Admin-created users only (no self sign-up)
- Q: Service exposure scope → A: Private network/VPN only (no direct public internet exposure)
- Q: Shared project permissions → A: Shared users can edit (collaborative editing)
- Q: SSO requirement → A: No SSO (local accounts only)

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Use Vikunja for personal task management (Priority: P1)

As a home-lab user, I can access Vikunja in a web browser, sign in, and manage my tasks (projects, tasks, due dates, and completion) so that I can replace ad-hoc to-do notes with a consistent workflow.

**Why this priority**: This delivers the core value (task management) and validates that the service is usable end-to-end.

**Independent Test**: Can be fully tested by opening the service, signing in, creating a project and a task, and confirming the task remains available after signing out and back in.

**Acceptance Scenarios**:

1. **Given** the service is reachable, **When** the user signs in with valid credentials, **Then** the user sees their task list UI.
2. **Given** the user is signed in, **When** the user creates a new project and a new task, **Then** the created items are visible in the UI.
3. **Given** the user has created tasks, **When** the user signs out and signs in again, **Then** the previously created items are still present.

---

### User Story 2 - Collaborate on shared projects (Priority: P2)

As a household member, I can share a project with another user and coordinate task ownership so that multiple people can collaborate without duplicating effort.

**Why this priority**: Collaboration is a key differentiator versus personal notes and enables shared household workflows.

**Independent Test**: Can be tested by creating two user accounts, sharing a project, and verifying both users can view and update tasks in that project.

**Acceptance Scenarios**:

1. **Given** two user accounts exist, **When** User A shares a project with User B, **Then** User B can access the shared project.
2. **Given** a project is shared, **When** User B edits or completes a task, **Then** User A can see the updated task state and content.

---

### User Story 3 - Operate the service safely (Priority: P3)

As the home-lab operator, I can maintain the Vikunja service (restart, upgrade, and recover) without losing task data so that the system remains trustworthy over time.

**Why this priority**: If data durability and recovery are not reliable, users will not adopt the system.

**Independent Test**: Can be tested by creating sample tasks, performing a controlled restart/upgrade, and validating that a restore procedure can recover the sample data from backups.

**Acceptance Scenarios**:

1. **Given** tasks and projects exist, **When** the service is restarted, **Then** the data remains intact after it comes back.
2. **Given** backups exist, **When** the operator performs a restore into a clean environment, **Then** the restored system contains the expected tasks and projects.

---

### Edge Cases

- Access is attempted from outside the trusted network/VPN: the service should not be reachable (or should deny access) by default.
- A user forgets credentials: there must be an operator-supported recovery path that does not expose other users' data.
- Storage becomes full or unavailable: the service should fail safely (no silent data loss) and surface an actionable error to the operator.
- Upgrade or configuration change fails: the service should allow rollback/recovery to a working state with data preserved.

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: The system MUST provide a web-based user interface for task management that is reachable from a supported client browser.
- **FR-002**: The system MUST require user authentication before showing any non-public data.
- **FR-003**: The system MUST support multiple user accounts and isolate each user's private data by default.
- **FR-004**: The system MUST allow users to create, update, and complete tasks within projects.
- **FR-005**: The system MUST allow users to share a project with another user and collaborate on tasks in that project.
- **FR-006**: The system MUST persist user data (users, projects, tasks, and related metadata) across restarts and upgrades.
- **FR-007**: The system MUST encrypt traffic in transit for interactive use (browser access).
- **FR-008**: The system MUST produce operator-accessible logs sufficient to diagnose common failures (startup failure, authentication failures, and data persistence issues).
- **FR-009**: The system MUST support automated backups of all user data at least once per day.
- **FR-010**: The system MUST support a documented restore procedure that can recover user data from backups.
- **FR-011**: The system MUST NOT allow public self sign-up; user accounts MUST be created by the operator/admin.
- **FR-012**: The system MUST be accessible only from the trusted network/VPN and MUST NOT be directly reachable from the public internet.
- **FR-013**: When a project is shared with another user, that user MUST be able to create, edit, and complete tasks within that shared project.

**Assumptions & Dependencies**:

- The primary access method is via a private, trusted network or VPN (not directly exposed to the public internet).
- Users are provisioned by the operator/admin (no self-registration).
- Initial rollout targets a small number of users (e.g., a household) and does not require SSO/enterprise identity integration for MVP.
- Backup storage is available and managed as part of the broader home-lab environment.

### Key Entities *(include if feature involves data)*

- **User**: An authenticated person with credentials and access permissions.
- **Project**: A container for tasks, which can be private or shared with other users.
- **Task**: A unit of work with attributes such as title, description, due date, and completion state.
- **Share/Permission**: The relationship that grants another user access to a project.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: A user can sign in and create a project and a task within 2 minutes on a home network.
- **SC-002**: After a restart, 100% of sampled tasks and projects created during testing remain present and correct.
- **SC-003**: A restore test can recover at least the last 24 hours of created tasks and projects (as demonstrated by a documented restore run).
- **SC-004**: During a 30-day period after rollout, the service is available for interactive use at least 99% of the time during expected usage hours.
