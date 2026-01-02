# Data Model: Gateway API CRD Installation

**Feature**: 004-gateway-api
**Date**: 2026-01-03

## Overview

This feature installs Gateway API CRDs into the Kubernetes cluster. The data model describes the Flux resources used for GitOps management and the resulting CRDs.

## Flux Resources

### GitRepository

Represents the upstream kubernetes-sigs/gateway-api repository.

| Field | Type | Description |
|-------|------|-------------|
| name | string | `gateway-api` |
| namespace | string | `flux-system` |
| url | string | `https://github.com/kubernetes-sigs/gateway-api` |
| ref.tag | string | `v1.4.1` (pinned version) |
| interval | duration | `1h` (check for updates hourly) |

### Kustomization (Flux)

Manages the CRD installation from the GitRepository.

| Field | Type | Description |
|-------|------|-------------|
| name | string | `gateway-api-crds` |
| namespace | string | `flux-system` |
| sourceRef | object | References `gateway-api` GitRepository |
| path | string | `./config/crd/standard` |
| interval | duration | `10m0s` |
| prune | boolean | `false` (CRDs should not be pruned) |

## Installed CRDs (Standard Channel)

The following CRDs are installed from Gateway API v1.4.1 standard channel:

### GatewayClass

Defines a class of Gateways with shared configuration.

| Field | Type | Description |
|-------|------|-------------|
| apiVersion | string | `gateway.networking.k8s.io/v1` |
| kind | string | `GatewayClass` |
| spec.controllerName | string | Controller that manages this class (e.g., `tailscale.com/ts-gateway`) |

### Gateway

Represents a load balancer or proxy that handles traffic.

| Field | Type | Description |
|-------|------|-------------|
| apiVersion | string | `gateway.networking.k8s.io/v1` |
| kind | string | `Gateway` |
| spec.gatewayClassName | string | Reference to GatewayClass |
| spec.listeners | array | List of listener configurations |
| spec.listeners[].name | string | Unique name for the listener |
| spec.listeners[].protocol | string | Protocol (HTTP, HTTPS, TLS, TCP, UDP) |
| spec.listeners[].port | integer | Port number |
| spec.listeners[].allowedRoutes | object | Route attachment rules |

### HTTPRoute

Defines HTTP routing rules to backend services.

| Field | Type | Description |
|-------|------|-------------|
| apiVersion | string | `gateway.networking.k8s.io/v1` |
| kind | string | `HTTPRoute` |
| spec.parentRefs | array | References to parent Gateways |
| spec.hostnames | array | Hostnames for routing |
| spec.rules | array | Routing rules |
| spec.rules[].matches | array | Match conditions (path, headers, etc.) |
| spec.rules[].backendRefs | array | Backend service references |

### ReferenceGrant

Allows cross-namespace references between resources.

| Field | Type | Description |
|-------|------|-------------|
| apiVersion | string | `gateway.networking.k8s.io/v1beta1` |
| kind | string | `ReferenceGrant` |
| spec.from | array | Source resources allowed to reference |
| spec.to | array | Target resources that can be referenced |

### GRPCRoute

Defines gRPC routing rules (GA in v1.2.0+).

| Field | Type | Description |
|-------|------|-------------|
| apiVersion | string | `gateway.networking.k8s.io/v1` |
| kind | string | `GRPCRoute` |
| spec.parentRefs | array | References to parent Gateways |
| spec.hostnames | array | Hostnames for routing |
| spec.rules | array | gRPC routing rules |

## Resource Relationships

```
┌─────────────────────────────────────────────────────────────┐
│                      flux-system namespace                   │
│                                                              │
│  ┌──────────────────┐      ┌─────────────────────────────┐  │
│  │  GitRepository   │◄─────│  Kustomization              │  │
│  │  gateway-api     │      │  gateway-api-crds           │  │
│  │                  │      │  path: ./config/crd/standard│  │
│  └────────┬─────────┘      └─────────────────────────────┘  │
│           │                                                  │
│           │ references                                       │
│           ▼                                                  │
│  ┌──────────────────────────────────────────────────────┐   │
│  │  https://github.com/kubernetes-sigs/gateway-api      │   │
│  │  tag: v1.4.1                                         │   │
│  └──────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────┘

                         installs
                            │
                            ▼
┌─────────────────────────────────────────────────────────────┐
│                    Cluster-scoped CRDs                       │
│                                                              │
│  ┌────────────────┐  ┌────────────────┐  ┌────────────────┐ │
│  │  GatewayClass  │  │    Gateway     │  │   HTTPRoute    │ │
│  └────────────────┘  └────────────────┘  └────────────────┘ │
│                                                              │
│  ┌────────────────┐  ┌────────────────┐                     │
│  │ ReferenceGrant │  │   GRPCRoute    │                     │
│  └────────────────┘  └────────────────┘                     │
└─────────────────────────────────────────────────────────────┘
```

## State Transitions

### GitRepository States

| State | Description |
|-------|-------------|
| Fetching | Downloading repository content |
| Ready | Repository content available |
| Failed | Unable to fetch repository |

### Kustomization States

| State | Description |
|-------|-------------|
| Reconciling | Applying resources to cluster |
| Ready | All resources applied successfully |
| Failed | Error applying resources |

### CRD States

| State | Description |
|-------|-------------|
| Established | CRD is registered and accepting resources |
| NamesAccepted | CRD names are valid and accepted |
