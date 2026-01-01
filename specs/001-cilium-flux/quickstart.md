# Quickstart: Manage Cilium with Flux

This quickstart is for operators validating the GitOps-managed Cilium setup.

## Prerequisites

- Flux is bootstrapped and reconciling the repository (Flux system healthy).
- You have `kubectl` access to the cluster.

## Validate Flux is healthy

```bash
kubectl -n flux-system get pods
kubectl -n flux-system get kustomizations
```

## Validate Cilium is Flux-managed and healthy

```bash
# Flux sources and releases
kubectl -n flux-system get helmrepositories
kubectl -n kube-system get helmreleases

# Cilium pods
kubectl -n kube-system get pods -l app.kubernetes.io/part-of=cilium

# Node readiness
kubectl get nodes
```

## Rollback workflow (high level)

1. Revert the Git commit that changed Cilium configuration.
2. Let Flux reconcile (or trigger reconciliation).
3. Verify Cilium and node readiness again using the commands above.
