# Feature Specification: Manage Cilium with Flux

**Feature Branch**: `001-cilium-flux`
**Created**: 2026-01-02
**Status**: Draft
**Input**: User description: "ciliumがhelm経由で手動でインストールされている。これをflux管理に移す。"

## Clarifications

### Session 2026-01-02

- Q: What is the Cilium upgrade policy after migration? → A: Pin version/config; changes via PR only.
- Q: What level of networking disruption is acceptable during migration? → A: No downtime/packet loss tolerated (target: zero).

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Declarative Cilium installation (Priority: P1)

As a cluster operator, I want Cilium to be installed and updated via Flux (GitOps) so that the cluster networking stack is reproducible, auditable, and recoverable without manual Helm actions.

**Why this priority**: Networking is foundational; if Cilium is not reliably present and consistent, the cluster cannot function.

**Independent Test**: A fresh environment/cluster (or a reset environment) can be brought to a working networking state by applying the Git repository state and letting Flux reconcile, without running any manual Helm install/upgrade commands.

**Acceptance Scenarios**:

1. **Given** a cluster where Cilium is currently installed manually, **When** the repository configuration is updated to have Flux manage Cilium, **Then** Flux reconciliation results in Cilium being present and healthy, without requiring manual Helm operations.
2. **Given** the Git repository state, **When** Flux reconciles from scratch (e.g., after a reinstall/recovery workflow), **Then** Cilium is installed as part of reconciliation and the cluster reaches a functional networking state.

---

### User Story 2 - Safe transition from manual management (Priority: P2)

As a cluster operator, I want the migration to avoid networking downtime and to prevent “double-management” conflicts, so that workloads keep running during the transition.

**Why this priority**: A broken CNI or conflicting controllers can cause immediate outages.

**Independent Test**: During the cutover window, there is at most one active source of truth managing Cilium, and Cilium remains ready throughout.

**Acceptance Scenarios**:

1. **Given** the cluster is running workloads, **When** the migration steps are followed, **Then** there is no observed loss of basic connectivity (target: zero downtime/packet loss).
2. **Given** the migration is complete, **When** the operator attempts to apply “manual Helm changes” out-of-band, **Then** Flux reconciliation restores the declared desired state (or the process clearly documents that out-of-band changes are unsupported).

---

### User Story 3 - Predictable upgrades and rollback (Priority: P3)

As a cluster operator, I want Cilium version/config changes to be done via pull requests only (no automatic upgrades), with an easy rollback path, so that changes are reviewable and reversable.

**Why this priority**: Controlled change management reduces risk and accelerates incident recovery.

**Independent Test**: A version/config change can be introduced via a PR, observed in the cluster after reconciliation, and reverted by reverting the PR.

**Acceptance Scenarios**:

1. **Given** a pull request that changes declared Cilium configuration, **When** it is merged, **Then** the cluster converges to the new desired state through Flux reconciliation.
2. **Given** a problematic change has been merged, **When** the change is reverted in Git, **Then** the cluster converges back to the previous desired state.

---

### Edge Cases

- What happens when the cluster has remnants of a prior manual Helm release that conflict with Flux-managed resources?
- How does the system handle a failed reconciliation (e.g., invalid configuration) without leaving the cluster in a partially-managed or broken state?
- What happens when the cluster is bootstrapped and the CNI is required early (ordering/bootstrapping constraints)?

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: The system MUST define Cilium installation and lifecycle management declaratively in Git, and it MUST be applied by Flux without manual Helm commands.
- **FR-002**: The system MUST avoid “double management” (manual Helm + Flux) of the same Cilium release/resources after migration completion.
- **FR-003**: The system MUST provide an operator-readable migration procedure that is safe to execute and includes verification steps.
- **FR-004**: The system MUST preserve existing cluster networking behavior as the default outcome of the migration (no intentional feature changes unless explicitly documented).
- **FR-005**: The system MUST support safe rollback to the previously working state using Git history (e.g., revert).
- **FR-006**: The system MUST NOT perform automatic Cilium version/config upgrades; all changes MUST be introduced via pull requests.
- **FR-007**: The migration procedure MUST target zero observed networking downtime/packet loss for existing workloads.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: After migration, a reconciled cluster reaches “Cilium healthy/ready” state with 0 manual Helm install/upgrade actions required.
- **SC-002**: A full disaster-recovery style reconciliation (from Git + Flux) results in a functional networking state within 30 minutes of starting reconciliation.
- **SC-003**: A Cilium configuration change can be rolled back by reverting Git changes, with the cluster returning to the previous working state within 15 minutes.
- **SC-004**: Over a 30-day period after migration, there are 0 incidents caused by drift between declared Cilium state and actual cluster state (as measured by repeated reconciliations converging without manual intervention).
- **SC-005**: During the migration window, there are 0 observed connectivity failures attributable to the Cilium management cutover.
