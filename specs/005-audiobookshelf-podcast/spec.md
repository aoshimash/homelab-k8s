# Feature Specification: Audiobookshelf Podcast Server

**Feature Branch**: `005-audiobookshelf-podcast`
**Created**: 2026-01-03
**Status**: Draft
**Input**: User description: "audiobookshelfを使ったpodcast配信サーバーを作成"

## Clarifications

### Session 2026-01-03

- Q: Audiobookshelfのデプロイ先namespaceは？ → A: `audiobookshelf`（専用namespace）
- Q: Tailscale Ingressのホスト名は？ → A: `audiobookshelf`（`audiobookshelf.<tailnet>.ts.net`でアクセス）
- Q: 永続ストレージの容量は？ → A: `50Gi`（ポッドキャストオーディオ用）
- Q: ProxyGroupの使用は？ → A: 既存のProxyGroup（`egress`）を使用（リソース効率優先）
- Q: デプロイメント方式は？ → A: 素のKubernetesマニフェスト（Deployment/Service/Ingress/PVC）

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Access podcast library via web interface (Priority: P1)

As a podcast listener, I want to access my podcast library through Audiobookshelf's web interface so that I can browse, search, and stream my podcast collection from any device on my tailnet.

**Why this priority**: Web access is the foundational user experience - users need to be able to access and manage their podcast library before any other functionality matters.

**Independent Test**: After deployment, the Audiobookshelf web UI is accessible via Tailscale Ingress, and users can browse the podcast library.

**Acceptance Scenarios**:

1. **Given** Audiobookshelf is deployed and configured, **When** a user accesses the Tailscale hostname from a device on the tailnet, **Then** the Audiobookshelf web interface is displayed.
2. **Given** the web interface is accessible, **When** a user logs in with valid credentials, **Then** they can browse the podcast library.
3. **Given** the user is logged in, **When** they select a podcast episode, **Then** they can stream the audio directly in the browser.

---

### User Story 2 - Subscribe to and download podcasts via RSS (Priority: P1)

As a podcast listener, I want Audiobookshelf to subscribe to podcast RSS feeds and automatically download new episodes so that I can build and maintain my podcast library without manual intervention.

**Why this priority**: Automatic podcast downloading is core functionality - without RSS subscription support, users would have to manually add each episode.

**Independent Test**: After adding a podcast RSS feed URL, new episodes are automatically discovered and downloaded to the library.

**Acceptance Scenarios**:

1. **Given** the Audiobookshelf web interface is accessible, **When** a user adds a podcast RSS feed URL, **Then** the podcast is added to the library and existing episodes are listed.
2. **Given** a podcast is subscribed, **When** a new episode is published to the RSS feed, **Then** Audiobookshelf detects and downloads the episode automatically within the configured check interval.
3. **Given** a podcast with multiple episodes, **When** viewing the podcast details, **Then** all episodes are listed with metadata (title, description, duration, publish date).

---

### User Story 3 - Persistent storage for podcast data (Priority: P1)

As a cluster operator, I want podcast data (audio files, metadata, user progress) to be stored persistently so that the library survives pod restarts and upgrades.

**Why this priority**: Data persistence is critical - users would lose their entire library and listening progress if storage is ephemeral.

**Independent Test**: After a pod restart or upgrade, all podcast data, user accounts, and listening progress remain intact.

**Acceptance Scenarios**:

1. **Given** Audiobookshelf is deployed with persistent storage, **When** the pod is restarted, **Then** all podcast files and metadata are preserved.
2. **Given** a user has listening progress on an episode, **When** the pod is upgraded to a new version, **Then** the listening progress is retained.
3. **Given** user accounts are configured, **When** the pod is deleted and recreated, **Then** user accounts and preferences persist.

---

### User Story 4 - GitOps-managed deployment (Priority: P2)

As a cluster operator, I want Audiobookshelf to be deployed and managed via GitOps (Flux) so that the deployment is version-controlled, reproducible, and follows the same pattern as other cluster infrastructure.

**Why this priority**: GitOps management ensures consistency with existing infrastructure patterns (Longhorn, Tailscale Operator, Grafana Alloy) and enables reproducible deployments.

**Independent Test**: After Flux reconciliation, Audiobookshelf is running and accessible without manual intervention.

**Acceptance Scenarios**:

1. **Given** the GitOps repository contains Audiobookshelf manifests, **When** Flux reconciles the cluster state, **Then** Audiobookshelf is deployed and running in the cluster.
2. **Given** a configuration change is committed to Git, **When** Flux reconciles, **Then** Audiobookshelf picks up the new configuration automatically.
3. **Given** the deployment manifests are present, **When** checking the cluster, **Then** all Audiobookshelf components (Deployment, Service, Ingress, PVC) are created.

---

### User Story 5 - Secure access via tailnet only (Priority: P2)

As a cluster operator, I want Audiobookshelf to be accessible only from within my tailnet so that my podcast library is not exposed to the public internet.

**Why this priority**: Security is important for a personal media server - the content should only be accessible to authorized devices on the tailnet.

**Independent Test**: The Audiobookshelf web interface is accessible from tailnet devices but returns connection refused from the public internet.

**Acceptance Scenarios**:

1. **Given** Audiobookshelf is exposed via Tailscale Ingress, **When** accessing from a device on the tailnet, **Then** the web interface is accessible.
2. **Given** Audiobookshelf is exposed via Tailscale Ingress, **When** attempting to access from outside the tailnet, **Then** the connection is refused or times out.
3. **Given** the Tailscale Ingress is configured, **When** accessing the service, **Then** TLS is automatically provisioned.

---

### Edge Cases

- What happens when the RSS feed URL is invalid or returns errors?
- How does the system behave when storage capacity is exhausted?
- What happens when a podcast episode download fails mid-way?
- How does the system handle podcast feeds with authentication requirements?
- What happens when the external network is temporarily unavailable during scheduled episode checks?
- How does the system behave when multiple users stream the same episode simultaneously?

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: The system MUST deploy Audiobookshelf via GitOps (Flux) workflow using raw Kubernetes manifests (Deployment/Service/Ingress/PVC) in the `audiobookshelf` namespace.
- **FR-002**: The system MUST expose Audiobookshelf web interface via Tailscale Ingress with hostname `audiobookshelf`, using the existing ProxyGroup (`egress`), accessible only from within the tailnet.
- **FR-003**: The system MUST provision persistent storage of `50Gi` for podcast audio files and metadata using the cluster's storage class (Longhorn).
- **FR-004**: The system MUST provision persistent storage for Audiobookshelf configuration and database.
- **FR-005**: The system MUST support podcast RSS feed subscription and automatic episode downloading.
- **FR-006**: The system MUST preserve all data (podcast files, metadata, user accounts, listening progress) across pod restarts and upgrades.
- **FR-007**: The system MUST automatically provision TLS certificates for the Tailscale Ingress endpoint.
- **FR-008**: The system MUST support web-based audio streaming for podcast playback.
- **FR-009**: The system SHOULD support configuration of podcast check intervals for new episode discovery.
- **FR-010**: The system SHOULD provide health check endpoints for monitoring pod status.

### Key Entities

- **Audiobookshelf**: Self-hosted audiobook and podcast server application that provides library management, streaming, and podcast subscription features.
- **Podcast Library**: Collection of podcast feeds subscribed by the user, containing episodes with audio files and metadata.
- **Podcast Episode**: Individual audio file with associated metadata (title, description, duration, publish date, playback progress).
- **User Account**: Authentication identity with associated preferences and listening progress across podcasts.
- **Persistent Volume**: Storage resources for audio files (`/podcasts`), metadata (`/metadata`), and configuration (`/config`).

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: The Audiobookshelf web interface is accessible via Tailscale hostname within 5 minutes of Flux reconciliation.
- **SC-002**: A podcast RSS feed can be added and episodes begin downloading within 10 minutes of subscription.
- **SC-003**: All podcast data persists across pod restarts, verified by checking library contents before and after restart.
- **SC-004**: TLS is automatically provisioned for the Tailscale Ingress endpoint, with HTTPS connections succeeding on first access.
- **SC-005**: The service is not accessible from outside the tailnet (connection refused or timeout when accessed from public internet).
- **SC-006**: Users can stream podcast episodes directly in the web browser with playback starting within 5 seconds.

## Assumptions

- The cluster has Longhorn configured as a storage class for persistent volumes (based on existing infrastructure).
- The Tailscale Operator with Ingress support is already deployed (spec 003).
- Users will create Audiobookshelf accounts through the web interface after initial deployment.
- External network access is available for downloading podcast episodes from RSS feed sources.
- The Audiobookshelf container image from Docker Hub or GitHub Container Registry is used.
