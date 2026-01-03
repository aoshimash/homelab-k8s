# Quickstart: Audiobookshelf Podcast Server

**Feature**: 005-audiobookshelf-podcast
**Date**: 2026-01-03

## Prerequisites

Before deploying Audiobookshelf, ensure:

- [ ] Flux CD is running and reconciling (`flux get kustomizations`)
- [ ] Longhorn is deployed and healthy (`kubectl get pods -n longhorn-system`)
- [ ] Tailscale Operator is running (`kubectl get pods -n tailscale`)
- [ ] ProxyGroup `ingress-proxies` exists (`kubectl get proxygroup -n tailscale`)

## Deployment Steps

### 1. Create the Flux Apps Kustomization (if not exists)

```bash
# Check if apps kustomization exists
flux get kustomizations apps

# If not, it will be created as part of this feature deployment
```

### 2. Deploy Audiobookshelf

The manifests are automatically applied by Flux when committed to Git:

```bash
# Verify deployment status
kubectl get all -n audiobookshelf

# Expected output:
# NAME                                READY   STATUS    RESTARTS   AGE
# pod/audiobookshelf-xxxxx            1/1     Running   0          5m
#
# NAME                    TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)   AGE
# service/audiobookshelf  ClusterIP   10.xx.xx.xx     <none>        80/TCP    5m
#
# NAME                             READY   UP-TO-DATE   AVAILABLE   AGE
# deployment.apps/audiobookshelf   1/1     1            1           5m
```

### 3. Verify Storage

```bash
# Check PVCs are bound
kubectl get pvc -n audiobookshelf

# Expected output:
# NAME                      STATUS   VOLUME   CAPACITY   ACCESS MODES   STORAGECLASS   AGE
# audiobookshelf-config     Bound    pvc-xxx  1Gi        RWO            longhorn       5m
# audiobookshelf-podcasts   Bound    pvc-xxx  50Gi       RWO            longhorn       5m
```

### 4. Verify Tailscale Ingress

```bash
# Check Ingress status
kubectl get ingress -n audiobookshelf

# Expected output:
# NAME            CLASS       HOSTS          ADDRESS   PORTS     AGE
# audiobookshelf  tailscale   audiobookshelf           80, 443   5m

# Check in Tailscale admin console
# The hostname should appear at: https://login.tailscale.com/admin/machines
```

### 5. Access Audiobookshelf

From any device on your tailnet:

1. Open browser to: `https://audiobookshelf.<your-tailnet>.ts.net`
2. Create admin account on first access
3. Add podcast library:
   - Settings → Libraries → Add Library
   - Library Type: Podcast
   - Folder: `/podcasts`

## First-Time Setup

### Create Admin Account

On first access, you'll be prompted to create an admin account:

1. Enter username (e.g., `admin`)
2. Enter password
3. Click "Create Account"

### Add Podcast Library

1. Go to **Settings** (gear icon)
2. Click **Libraries** → **Add Library**
3. Configure:
   - **Name**: Podcasts
   - **Library Type**: Podcast
   - **Folders**: `/podcasts`
4. Click **Save**

### Subscribe to a Podcast

1. Click the **+** button in the header
2. Select **Add Podcast by RSS Feed**
3. Enter the podcast RSS feed URL
4. Click **Search**
5. Configure download settings
6. Click **Add Podcast**

## Verification Checklist

| Check | Command/Action | Expected Result |
|-------|----------------|-----------------|
| Pod running | `kubectl get pods -n audiobookshelf` | 1/1 Running |
| Storage bound | `kubectl get pvc -n audiobookshelf` | Both Bound |
| Web UI accessible | Browser to `https://audiobookshelf.<tailnet>.ts.net` | Login page |
| Health check | `curl https://audiobookshelf.<tailnet>.ts.net/healthcheck` | `{"success":true}` |
| Add podcast | Add RSS feed in UI | Podcast appears in library |

## Troubleshooting

### Pod not starting

```bash
# Check pod events
kubectl describe pod -n audiobookshelf -l app=audiobookshelf

# Check logs
kubectl logs -n audiobookshelf -l app=audiobookshelf
```

### Storage issues

```bash
# Check Longhorn volumes
kubectl get volumes.longhorn.io -n longhorn-system

# Check PVC events
kubectl describe pvc -n audiobookshelf audiobookshelf-podcasts
```

### Ingress not working

```bash
# Check ProxyGroup
kubectl get proxygroup -n tailscale ingress-proxies -o yaml

# Check Tailscale Operator logs
kubectl logs -n tailscale -l app.kubernetes.io/name=tailscale-operator
```

### Cannot access from tailnet

1. Verify device is on the same tailnet
2. Check Tailscale admin console for the `audiobookshelf` device
3. Review ACL policies in Tailscale admin

## Useful Commands

```bash
# Force Flux reconciliation
flux reconcile kustomization apps --with-source

# Restart Audiobookshelf pod
kubectl rollout restart deployment/audiobookshelf -n audiobookshelf

# View Audiobookshelf logs
kubectl logs -f -n audiobookshelf -l app=audiobookshelf

# Port-forward for local testing (bypasses Tailscale)
kubectl port-forward -n audiobookshelf svc/audiobookshelf 8080:80
# Then access: http://localhost:8080
```
