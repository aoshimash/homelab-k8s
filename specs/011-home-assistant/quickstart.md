# Quickstart: Home Assistant on Kubernetes

**Feature**: 011-home-assistant
**Date**: 2026-01-04

## Prerequisites

- [ ] Kubernetes cluster running (Talos Linux)
- [ ] Flux CD installed and reconciling
- [ ] Longhorn storage available
- [ ] Tailscale operator deployed with `ingress-proxies` ProxyGroup
- [ ] SwitchBot Hub 2 with Matter enabled (via SwitchBot app)
- [ ] SwitchBot Hub 2 on same L2 network as Kubernetes node

## Quick Deploy

### 1. Create Directory Structure

```bash
mkdir -p k8s/apps/home-assistant/app
```

### 2. Create Namespace

```bash
cat > k8s/apps/home-assistant/namespace.yaml << 'EOF'
apiVersion: v1
kind: Namespace
metadata:
  name: home-assistant
EOF
```

### 3. Create PVC

```bash
cat > k8s/apps/home-assistant/app/pvc.yaml << 'EOF'
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: home-assistant-config
  namespace: home-assistant
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: longhorn
  resources:
    requests:
      storage: 5Gi
EOF
```

### 4. Create ConfigMap (Sample Automation)

```bash
cat > k8s/apps/home-assistant/app/configmap.yaml << 'EOF'
apiVersion: v1
kind: ConfigMap
metadata:
  name: home-assistant-automations
  namespace: home-assistant
data:
  automations.yaml: |
    - id: startup_notification
      alias: "Startup Notification"
      trigger:
        - platform: homeassistant
          event: start
      action:
        - service: persistent_notification.create
          data:
            title: "Home Assistant Started"
            message: "Home Assistant has started successfully."
EOF
```

### 5. Create Deployment

```bash
cat > k8s/apps/home-assistant/app/deployment.yaml << 'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: home-assistant
  namespace: home-assistant
spec:
  replicas: 1
  selector:
    matchLabels:
      app: home-assistant
  template:
    metadata:
      labels:
        app: home-assistant
    spec:
      hostNetwork: true
      dnsPolicy: ClusterFirstWithHostNet
      nodeSelector:
        kubernetes.io/hostname: homelab-node-01
      containers:
        - name: home-assistant
          image: ghcr.io/home-assistant/home-assistant:2024.12
          ports:
            - containerPort: 8123
          volumeMounts:
            - name: config
              mountPath: /config
            - name: automations
              mountPath: /config/automations.yaml
              subPath: automations.yaml
          resources:
            requests:
              cpu: 100m
              memory: 512Mi
            limits:
              cpu: 2000m
              memory: 2Gi
          livenessProbe:
            httpGet:
              path: /
              port: 8123
            initialDelaySeconds: 60
            periodSeconds: 30
          readinessProbe:
            httpGet:
              path: /
              port: 8123
            initialDelaySeconds: 30
            periodSeconds: 10
      volumes:
        - name: config
          persistentVolumeClaim:
            claimName: home-assistant-config
        - name: automations
          configMap:
            name: home-assistant-automations
EOF
```

### 6. Create Service

```bash
cat > k8s/apps/home-assistant/app/service.yaml << 'EOF'
apiVersion: v1
kind: Service
metadata:
  name: home-assistant
  namespace: home-assistant
spec:
  type: ClusterIP
  selector:
    app: home-assistant
  ports:
    - port: 80
      targetPort: 8123
      protocol: TCP
EOF
```

### 7. Create Ingress

```bash
cat > k8s/apps/home-assistant/app/ingress.yaml << 'EOF'
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: home-assistant
  namespace: home-assistant
  annotations:
    tailscale.com/proxy-group: ingress-proxies
spec:
  ingressClassName: tailscale
  defaultBackend:
    service:
      name: home-assistant
      port:
        number: 80
  tls:
    - hosts:
        - home-assistant
EOF
```

### 8. Create Kustomizations

```bash
# App kustomization
cat > k8s/apps/home-assistant/app/kustomization.yaml << 'EOF'
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - pvc.yaml
  - configmap.yaml
  - deployment.yaml
  - service.yaml
  - ingress.yaml
EOF

# Root kustomization
cat > k8s/apps/home-assistant/kustomization.yaml << 'EOF'
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - namespace.yaml
  - app
EOF
```

### 9. Update Apps Kustomization

Add `home-assistant` to `k8s/apps/kustomization.yaml`:

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - audiobookshelf
  - home-assistant  # Add this line
```

### 10. Commit and Push

```bash
git add k8s/apps/home-assistant k8s/apps/kustomization.yaml
git commit -m "feat: add Home Assistant deployment with Matter support"
git push
```

### 11. Wait for Reconciliation

```bash
# Watch Flux reconciliation
flux get kustomizations -w

# Check deployment status
kubectl get all -n home-assistant
```

## Post-Deployment Setup

### Access Home Assistant

1. Open browser: `https://home-assistant.<your-tailnet>.ts.net`
2. Complete onboarding wizard:
   - Create admin account
   - Set location (for timezone/weather)
   - Configure integrations

### Add Matter Integration

1. Go to **Settings** → **Devices & Services**
2. Click **Add Integration**
3. Search for **Matter**
4. Click **Configure**
5. Matter controller will initialize

### Pair SwitchBot Hub 2

1. In SwitchBot app:
   - Open Hub 2 settings
   - Enable Matter
   - Get pairing code (11-digit code)

2. In Home Assistant:
   - Go to **Settings** → **Devices & Services** → **Matter**
   - Click **Add Device**
   - Enter pairing code
   - Wait for commissioning (may take 1-2 minutes)

3. Verify:
   - Hub 2 appears in Matter devices
   - Connected devices (e.g., lights, sensors) are available

## Verification Commands

```bash
# Check pod is running on correct node
kubectl get pod -n home-assistant -o wide

# Check PVC is bound
kubectl get pvc -n home-assistant

# Check logs
kubectl logs -n home-assistant -l app=home-assistant -f

# Check Tailscale ingress
kubectl get ingress -n home-assistant

# Test connectivity
curl -I https://home-assistant.<your-tailnet>.ts.net
```

## Troubleshooting

### Pod not starting

```bash
# Check events
kubectl describe pod -n home-assistant -l app=home-assistant

# Check PVC status
kubectl get pvc -n home-assistant
```

### Matter devices not discovered

1. Verify hostNetwork is enabled:
   ```bash
   kubectl get pod -n home-assistant -o jsonpath='{.items[0].spec.hostNetwork}'
   # Should return: true
   ```

2. Check mDNS is working:
   ```bash
   # On the node (if accessible)
   avahi-browse -a
   ```

3. Verify Hub 2 is on same network as node

### ConfigMap changes not applied

Home Assistant requires manual reload:
1. UI: **Settings** → **Automations** → **Reload**
2. Or restart pod:
   ```bash
   kubectl rollout restart deployment/home-assistant -n home-assistant
   ```
