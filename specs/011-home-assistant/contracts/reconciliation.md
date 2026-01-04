# Reconciliation Contract: Home Assistant

**Feature**: 011-home-assistant
**Date**: 2026-01-04

## Overview

This document defines the expected reconciliation behavior for Home Assistant deployment via Flux GitOps.

## Flux Kustomization Hierarchy

```
k8s/flux/apps-kustomization.yaml
    └── k8s/apps/kustomization.yaml
        └── k8s/apps/home-assistant/kustomization.yaml
            ├── namespace.yaml
            └── app/kustomization.yaml
                ├── deployment.yaml
                ├── service.yaml
                ├── ingress.yaml
                ├── pvc.yaml
                └── configmap.yaml
```

## Reconciliation Scenarios

### Scenario 1: Initial Deployment

**Trigger**: First commit of home-assistant manifests to main branch

**Expected Behavior**:
1. Flux detects new resources in `k8s/apps/home-assistant/`
2. Namespace `home-assistant` created
3. PVC `home-assistant-config` created, Longhorn provisions volume
4. ConfigMap `home-assistant-automations` created
5. Deployment `home-assistant` created
6. Pod scheduled on `homelab-node-01` (nodeSelector)
7. Service `home-assistant` created
8. Ingress `home-assistant` created, Tailscale operator provisions proxy

**Verification**:
```bash
# Check namespace
kubectl get namespace home-assistant

# Check all resources
kubectl get all,pvc,configmap,ingress -n home-assistant

# Check pod is on correct node
kubectl get pod -n home-assistant -o wide

# Check Tailscale proxy
kubectl get pods -n tailscale -l tailscale.com/proxy-group=ingress-proxies
```

**Success Criteria**:
- Pod status: Running
- PVC status: Bound
- Ingress has assigned hostname
- Access via `https://home-assistant.<tailnet>.ts.net` returns Home Assistant UI

---

### Scenario 2: ConfigMap Update (Automation Change)

**Trigger**: Update to `configmap.yaml` with new automation

**Expected Behavior**:
1. Flux detects ConfigMap change
2. ConfigMap updated in cluster
3. Pod continues running (no automatic restart)
4. User triggers Home Assistant reload via UI or API

**Verification**:
```bash
# Check ConfigMap content
kubectl get configmap home-assistant-automations -n home-assistant -o yaml

# Verify automation loaded in Home Assistant
# Access Home Assistant UI → Settings → Automations
```

**Note**: Home Assistant does not auto-reload ConfigMap changes. Options:
- Manual reload via UI: Settings → Automations → Reload
- API call: `POST /api/services/automation/reload`
- Pod restart (if acceptable): `kubectl rollout restart deployment/home-assistant -n home-assistant`

---

### Scenario 3: Image Update

**Trigger**: Update to `deployment.yaml` with new image tag

**Expected Behavior**:
1. Flux detects Deployment change
2. New ReplicaSet created
3. Old pod terminated gracefully
4. New pod scheduled on `homelab-node-01`
5. Matter devices remain connected (credentials in PVC)

**Verification**:
```bash
# Watch rollout
kubectl rollout status deployment/home-assistant -n home-assistant

# Check new image
kubectl get deployment home-assistant -n home-assistant -o jsonpath='{.spec.template.spec.containers[0].image}'

# Verify Matter devices still connected
# Access Home Assistant UI → Settings → Devices & Services → Matter
```

**Success Criteria**:
- New pod running with updated image
- Home Assistant UI accessible
- Matter devices operational (no re-pairing needed)

---

### Scenario 4: PVC Resize

**Trigger**: Update to `pvc.yaml` with increased storage request

**Expected Behavior**:
1. Flux detects PVC change
2. Longhorn expands volume (online expansion supported)
3. No pod restart required

**Verification**:
```bash
# Check PVC capacity
kubectl get pvc home-assistant-config -n home-assistant

# Verify inside pod
kubectl exec -n home-assistant deployment/home-assistant -- df -h /config
```

---

### Scenario 5: Node Failure Recovery

**Trigger**: `homelab-node-01` becomes unavailable

**Expected Behavior**:
1. Pod enters Pending state (nodeSelector constraint)
2. Pod does NOT reschedule to another node
3. Home Assistant and Matter devices unavailable
4. When node recovers, pod restarts on same node
5. Matter devices reconnect automatically

**Verification**:
```bash
# Check pod status
kubectl get pod -n home-assistant

# Check events
kubectl describe pod -n home-assistant -l app=home-assistant
```

**Note**: This is expected behavior for single-node cluster with Matter devices. High availability would require different architecture (out of scope).

---

### Scenario 6: Drift Correction

**Trigger**: Manual `kubectl` change to resources

**Expected Behavior**:
1. Flux detects drift from Git state
2. Resources reconciled back to Git-defined state
3. Manual changes reverted

**Verification**:
```bash
# Check Flux reconciliation status
flux get kustomizations

# Force reconciliation
flux reconcile kustomization apps --with-source
```

## Health Checks

### Deployment Health

| Check | Method | Expected |
|-------|--------|----------|
| Pod Running | `kubectl get pod -n home-assistant` | 1/1 Running |
| Liveness Probe | HTTP GET :8123/ | 200 OK |
| Readiness Probe | HTTP GET :8123/ | 200 OK |
| Node Placement | `kubectl get pod -o wide` | homelab-node-01 |

### Storage Health

| Check | Method | Expected |
|-------|--------|----------|
| PVC Bound | `kubectl get pvc -n home-assistant` | Bound |
| Volume Healthy | Longhorn UI | Healthy |
| Data Persisted | Pod restart test | Config retained |

### Network Health

| Check | Method | Expected |
|-------|--------|----------|
| Service Endpoints | `kubectl get endpoints -n home-assistant` | 1 endpoint |
| Ingress Ready | `kubectl get ingress -n home-assistant` | ADDRESS assigned |
| External Access | `curl https://home-assistant.<tailnet>.ts.net` | 200 OK |
| Matter/mDNS | Home Assistant UI | Devices discovered |

## Rollback Procedure

If deployment fails:

```bash
# Check Flux status
flux get kustomizations

# View recent commits
git log --oneline -5

# Revert to previous commit
git revert HEAD
git push

# Force reconciliation
flux reconcile kustomization apps --with-source

# Or manual rollback
kubectl rollout undo deployment/home-assistant -n home-assistant
```

## Monitoring Integration

Future integration with Grafana Alloy (out of scope for this feature):

- Metrics endpoint: `http://home-assistant:8123/api/prometheus`
- Requires Home Assistant Prometheus integration enabled
- ServiceMonitor resource for automatic scraping
