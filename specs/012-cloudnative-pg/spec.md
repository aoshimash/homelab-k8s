# Feature Specification: CloudNativePG PostgreSQL Cluster

**Feature Branch**: `012-cloudnative-pg`  
**Created**: 2026-01-07  
**Status**: Draft  
**Input**: User description: "PostgreSQLクラスタ作成 https://github.com/aoshimash/homelab-k8s/issues/79"

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Shared PostgreSQL Database Cluster (Priority: P1)

As a homelab operator, I want a shared PostgreSQL database cluster running in Kubernetes so that multiple applications can use PostgreSQL without deploying separate database instances for each application.

**Why this priority**: This is the core functionality that enables all other use cases. Without the shared cluster, no applications can use PostgreSQL.

**Independent Test**: Deploy the PostgreSQL cluster and verify it is running and accepting connections from within the cluster using the default superuser credentials.

**Acceptance Scenarios**:

1. **Given** the Kubernetes cluster is running with Flux, **When** the CloudNativePG operator and cluster manifests are deployed, **Then** a PostgreSQL cluster should be created and reach a healthy state.
2. **Given** the PostgreSQL cluster is running, **When** an administrator connects using the provided credentials, **Then** they should be able to execute SQL queries successfully.
3. **Given** the PostgreSQL cluster is running, **When** checking the cluster status, **Then** the cluster should show as healthy with at least one running instance.

---

### User Story 2 - Application Database Provisioning (Priority: P2)

As an application developer, I want to provision a dedicated database and user for my application so that my application data is isolated from other applications sharing the same PostgreSQL cluster.

**Why this priority**: Applications need their own databases/users to function. This enables the multi-tenant usage pattern described in the requirements.

**Independent Test**: Create a new database and user, then verify an application can connect using the application-specific credentials and access only its own database.

**Acceptance Scenarios**:

1. **Given** the PostgreSQL cluster is running, **When** a new database and user are created for an application, **Then** the application should be able to connect using the new credentials.
2. **Given** an application has a dedicated database, **When** connecting with application credentials, **Then** the application should not have access to other application databases.
3. **Given** a database connection request, **When** an application uses the cluster's internal DNS name (e.g., `postgres-cluster-rw.postgres.svc.cluster.local`), **Then** the connection should succeed.

---

### User Story 3 - Persistent Data Storage (Priority: P3)

As a homelab operator, I want PostgreSQL data to persist across pod restarts and node reboots so that application data is not lost during maintenance or unexpected failures.

**Why this priority**: Data persistence is critical for production use, but the cluster can function temporarily without it for testing purposes.

**Independent Test**: Create data in the database, restart the PostgreSQL pod, and verify the data is still present after restart.

**Acceptance Scenarios**:

1. **Given** data has been written to a PostgreSQL database, **When** the PostgreSQL pod is restarted, **Then** the data should be preserved and accessible.
2. **Given** the PostgreSQL cluster is deployed, **When** checking storage configuration, **Then** the cluster should be using Longhorn-backed persistent volumes.

---

### User Story 4 - Automated Backups (Priority: P4)

As a homelab operator, I want automated backups of the PostgreSQL cluster to external storage so that I can recover from data loss or corruption.

**Why this priority**: While important for production readiness, the cluster can operate without backups initially.

**Independent Test**: Configure backup settings and verify that backups are created successfully to the Longhorn R2 backup target.

**Acceptance Scenarios**:

1. **Given** backup is configured for the PostgreSQL cluster, **When** a scheduled backup runs, **Then** the backup should complete successfully and be stored in the backup target.
2. **Given** a backup exists, **When** needed for disaster recovery, **Then** the backup should be restorable to a new cluster instance.

---

### Edge Cases

- What happens when the PostgreSQL pod runs out of disk space? The cluster should emit alerts/events before reaching critical capacity.
- How does the system handle connection failures? Applications should implement connection retry logic; the cluster provides read-write and read-only service endpoints for high availability scenarios.
- What happens if Longhorn storage becomes unavailable? The PostgreSQL pod should remain in pending state until storage is restored.

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: System MUST deploy CloudNativePG operator to the Kubernetes cluster via Flux HelmRelease.
- **FR-002**: System MUST create a PostgreSQL cluster in a dedicated namespace (e.g., `postgres`).
- **FR-012**: System MUST deploy PostgreSQL version 16 as the database engine.
- **FR-003**: System MUST use Longhorn for persistent volume storage.
- **FR-004**: System MUST provide internal DNS endpoint for read-write connections (e.g., `*-rw.postgres.svc.cluster.local`).
- **FR-005**: System MUST store database superuser credentials in a Kubernetes Secret encrypted with SOPS.
- **FR-006**: System MUST support single-instance deployment suitable for homelab (single-node cluster with replica count of 1).
- **FR-009**: System MUST allocate 10GB of persistent storage for the PostgreSQL cluster.
- **FR-007**: System MUST integrate with existing Longhorn R2 backup infrastructure for database backups.
- **FR-010**: System MUST perform automated database backups daily (once per day).
- **FR-011**: System MUST retain database backups for 7 days before automatic deletion.
- **FR-008**: System MUST follow existing repository patterns for Flux manifests (namespace, helmrepository, helmrelease, kustomization).

### Key Entities

- **CloudNativePG Operator**: The Kubernetes operator that manages PostgreSQL cluster lifecycle.
- **PostgreSQL Cluster**: The custom resource that defines the PostgreSQL database cluster configuration.
- **Database Credentials**: Kubernetes Secret containing superuser username and password for cluster administration.
- **Backup Configuration**: Settings for automated database backups to external storage.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: PostgreSQL cluster reaches healthy state within 5 minutes of manifest deployment.
- **SC-002**: Applications can connect to PostgreSQL using the internal DNS endpoint within 1 minute of cluster becoming healthy.
- **SC-003**: Data written to PostgreSQL survives pod restarts with zero data loss.
- **SC-004**: Database backups complete successfully within 10 minutes of scheduled time.
- **SC-005**: Administrators can create new databases and users for applications within 2 minutes using standard PostgreSQL commands.

## Clarifications

### Session 2026-01-07

- Q: PostgreSQLストレージサイズ → A: 10GB（中規模、複数アプリの共有DB向け）
- Q: バックアップ頻度 → A: 日次（1日1回）
- Q: バックアップ保持期間 → A: 7日間（1週間）
- Q: PostgreSQLバージョン → A: PostgreSQL 16（現在の安定版）

## Assumptions

- The Kubernetes cluster is already running with Flux CD for GitOps management.
- Longhorn storage system is already deployed and operational.
- SOPS encryption is already configured for secrets management.
- Single-node deployment is acceptable for homelab use case (no high availability requirements).
- Longhorn R2 backup target is already configured and operational.
