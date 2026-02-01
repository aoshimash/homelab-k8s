# Feature Specification: Deploy Actions Runner Controller (ARC)

**Feature Branch**: `016-deploy-arc`  
**Created**: 2026-02-01  
**Status**: Draft  
**Input**: User description: "https://github.com/actions/actions-runner-controller をデプロイ"

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Run GitHub Actions Workflows on Self-Hosted Runners (Priority: P1)

As a developer, I want to run GitHub Actions workflows on my homelab Kubernetes cluster so that I can execute CI/CD pipelines using my own infrastructure without relying on GitHub-hosted runners.

**Why this priority**: This is the core functionality of ARC. Without self-hosted runners being operational, no other features can be utilized. It provides immediate value by enabling private CI/CD execution.

**Independent Test**: Can be fully tested by triggering a simple workflow that uses `runs-on: self-hosted` and verifying it completes successfully on the homelab cluster.

**Acceptance Scenarios**:

1. **Given** ARC is deployed to the homelab cluster, **When** a GitHub Actions workflow specifies the configured runner label, **Then** the workflow job is picked up and executed by a runner pod in the cluster.
2. **Given** ARC is running, **When** a workflow job completes (success or failure), **Then** the runner pod terminates cleanly and resources are released.
3. **Given** ARC is deployed, **When** no workflow jobs are pending, **Then** runner pods scale down to the configured minimum (can be zero).

---

### User Story 2 - Automatic Runner Scaling (Priority: P2)

As a developer, I want runners to automatically scale based on workflow demand so that resources are used efficiently and workflows don't wait unnecessarily for available runners.

**Why this priority**: Autoscaling ensures efficient resource utilization in the homelab environment where resources are limited. It prevents resource waste when idle and provides capacity when needed.

**Independent Test**: Can be tested by triggering multiple concurrent workflows and observing that additional runner pods are created to handle the load.

**Acceptance Scenarios**:

1. **Given** minimum runners is configured to 0 and no jobs are queued, **When** a workflow is triggered, **Then** a runner pod is created within a reasonable time to execute the job.
2. **Given** multiple workflow jobs are queued, **When** demand exceeds current runner count, **Then** additional runner pods are created up to the configured maximum.
3. **Given** runner pods are running, **When** all jobs complete, **Then** runners scale down to the configured minimum after the idle timeout.

---

### User Story 3 - GitOps-Managed Deployment (Priority: P3)

As a cluster administrator, I want ARC to be managed through GitOps (FluxCD) so that all configuration changes are version-controlled, auditable, and consistent with the existing homelab infrastructure management approach.

**Why this priority**: Consistency with existing infrastructure patterns is important for maintainability but is not required for basic functionality.

**Independent Test**: Can be tested by making a configuration change via Git commit and verifying FluxCD reconciles the change to the cluster.

**Acceptance Scenarios**:

1. **Given** ARC manifests are in the Git repository, **When** FluxCD reconciles, **Then** ARC components are deployed to the cluster automatically.
2. **Given** ARC is deployed via FluxCD, **When** a configuration change is committed to the repository, **Then** FluxCD applies the change to the cluster.

---

### Edge Cases

- What happens when GitHub API is unreachable? Runners should gracefully handle temporary connectivity issues and retry.
- How does the system handle when a runner pod crashes mid-job? The job should be marked as failed and GitHub Actions should handle retry logic if configured.
- What happens when cluster resources are exhausted? New runner pods should remain pending until resources become available, preventing node overload.
- How does the system handle when authentication credentials expire? The system should provide clear error messages and not crash.

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: System MUST deploy ARC controller components to the homelab Kubernetes cluster
- **FR-002**: System MUST authenticate with GitHub using a GitHub App for automatic token refresh and enhanced security
- **FR-003**: System MUST create runner pods in Kubernetes mode (each job step runs in separate container, no Docker daemon required) when workflow jobs are queued targeting the configured runner labels
- **FR-004**: System MUST terminate runner pods after job completion (ephemeral runners)
- **FR-005**: System MUST scale runners between 0 (minimum) and 3 (maximum) based on demand
- **FR-006**: System MUST be deployable and manageable through FluxCD GitOps workflow
- **FR-007**: System MUST store authentication credentials securely using SOPS-encrypted secrets (consistent with existing homelab patterns)
- **FR-008**: System MUST configure runner label as `homelab` to identify the homelab runner pool (workflows use `runs-on: homelab`)
- **FR-009**: Runners MUST be registered at the GitHub organization level to allow sharing across all repositories within the organization

### Key Entities

- **Runner Controller**: The central component that manages runner lifecycle, communicates with GitHub API, and handles scaling decisions
- **Runner Scale Set**: A group of runners with shared configuration (labels, resource limits, scaling parameters)
- **Runner Pod**: An ephemeral Kubernetes pod that executes a single GitHub Actions job (resource limits: 4 CPU, 8GB RAM per pod)
- **Authentication Credentials**: GitHub App credentials (App ID, Installation ID, Private Key) used to register runners and receive job assignments

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: Workflow jobs targeting the self-hosted runner complete successfully at least 95% of the time (excluding workflow logic failures)
- **SC-002**: Runner pods are created and ready to accept jobs within 2 minutes of a workflow being triggered
- **SC-003**: Runner pods scale down to minimum count within 5 minutes of all jobs completing
- **SC-004**: ARC deployment is fully managed via Git commits with no manual kubectl commands required for normal operations
- **SC-005**: Authentication credentials are encrypted at rest and never exposed in plain text in the repository

## Clarifications

### Session 2026-02-01

- Q: GitHub authentication method? → A: GitHub App (automatic token refresh, enhanced security)
- Q: Runner scaling limits? → A: Min: 0, Max: 3 (balanced for homelab)
- Q: Runner resource limits per pod? → A: Large: 4 CPU, 8GB RAM (heavy builds)
- Q: Runner container mode? → A: Kubernetes mode (no Docker daemon, better security)
- Q: Runner label name? → A: `homelab` (workflows use `runs-on: homelab`)

## Assumptions

- The homelab Kubernetes cluster has sufficient resources (CPU, memory) to run the ARC controller and at least one runner pod
- GitHub API is accessible from the homelab network
- The target GitHub organization/repository allows self-hosted runners
- FluxCD is already operational in the cluster (confirmed by existing infrastructure)
- SOPS encryption is configured for secret management (confirmed by existing patterns in the repository)
