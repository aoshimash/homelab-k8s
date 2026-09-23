# local-path-provisioner - Local Storage on a Talos User Volume

[rancher/local-path-provisioner](https://github.com/rancher/local-path-provisioner)
provisions PersistentVolumes as plain directories on the node's disk. It is the
storage class that replaces Longhorn, as recorded in
[storage-migration-decision.md](storage-migration-decision.md). It was
introduced by [#302](https://github.com/aoshimash/homelab-k8s/issues/302).

There is no replication. A volume is a directory on one node, and it is lost if
that node's disk is lost. Recovery comes from the R2 backups taken by K8up
([k8up.md](k8up.md)), and only for volumes that opted in to them.

## Current Configuration

| Item | Value |
|------|-------|
| Chart | `deploy/chart/local-path-provisioner` from the upstream git repository, pinned by `ref.tag` in `k8s/infrastructure/local-path-provisioner/gitrepository.yaml` |
| Namespace | `local-path-storage` (Pod Security `privileged`) |
| StorageClass | `local-path` (provisioner `cluster.local/local-path-provisioner`) |
| Default class | **no**: Longhorn stays the default until the migration is complete |
| Reclaim policy | `Retain` for the duration of the migration |
| Binding mode | `WaitForFirstConsumer` |
| Root on the node | `/var/mnt/local-path-provisioner`, a Talos user volume of type `directory` |
| On-disk layout | `<namespace>/<claim>/<pv-name>/` under the root |
| Helper image | `docker.io/library/busybox`, pinned by tag and digest |
| Capacity | **not enforced**. A PVC's requested size is documentation only |

### Why these settings

- **A `directory` user volume, not a partition.** The node has one disk
  (`nvme0n1`, 1TB), and `EPHEMERAL` (`/var`) was provisioned across all of it
  (`nvme0n1p4`, 998GB), so no free space is left for a partition-type user
  volume. `EPHEMERAL` cannot be shrunk in place: Talos applies a `VolumeConfig`
  only when the volume is first provisioned. Re-provisioning it on a single node
  wipes etcd and every workload's data. A `directory` user volume is a directory
  on `EPHEMERAL`, which Talos bind-mounts at `/var/mnt/<name>` and propagates
  into the kubelet. The cost is that nothing isolates it: see
  [Capacity](#capacity).
- **Chart from git.** The project publishes no Helm repository; the chart exists
  only in the source repository under `deploy/chart/`. A Flux `GitRepository`
  fetches it (`sparseCheckout` limits the fetch to `deploy/chart`), and the
  HelmRelease references that source. This keeps the HelmRelease shape used by
  every other component. The alternative was the Kustomize remote base that
  Talos's own guide uses, which would configure the provisioner through JSON
  patches on a ConfigMap instead of chart values.
- **`reconcileStrategy: Revision`.** For a `GitRepository` source, Flux ignores
  `spec.chart.spec.version`. The pin is `ref.tag`, and `Revision` rebuilds the
  chart artifact whenever the fetched revision changes, so the chart that runs
  is the chart the tag points to. A git tag is a mutable pin, unlike a digest:
  if upstream force-moves a tag, the change is deployed without a reviewed
  commit here. The default `ChartVersion` strategy would not avoid that. It
  would only stop redeploying while the chart version stayed the same, and the
  cluster would then run something other than what the tag names. Pinning
  `ref.commit` instead would close the gap, but Renovate's flux manager tracks
  commits rather than tags when `commit` is set. The tag was chosen so version
  bumps still arrive as PRs.
- **Not the default class.** Longhorn owns every existing volume. Making this
  the default before any volume has moved would send new claims to a
  provisioner that has not been exercised yet.
- **`Retain`.** During the migration, deleting a claim by mistake must not
  delete its data. The trade-off is that a deleted claim leaves a `Released` PV
  and its directory behind, which have to be cleaned up by hand (see
  [Release a retained volume](#release-a-retained-volume)).
- **`WaitForFirstConsumer`.** This is the chart default. A volume is bound to
  the node it was created on. With this mode the volume is created on the node
  the consuming pod is scheduled to, which is what makes it behave correctly
  once a second node joins.
- **`<namespace>/<claim>/<pv-name>`.** Recovery may mean browsing the node's
  disk by hand, so the path has to say which claim a directory belongs to. The
  provisioner requires a custom pattern to start with `<namespace>/<claim>/`.
  The trailing PV name keeps each volume's directory unique. Under `Retain`, a
  claim that is deleted and recreated with the same name would otherwise be
  handed the previous claim's leftover data.
- **Pinned helper image.** The provisioner runs a short-lived helper pod to
  create and delete volume directories. The chart's default image for that pod
  is `busybox:latest`, a floating tag, which the
  [pinning policy](renovate.md#pinning-policy) does not allow.
- **Privileged namespace.** The helper pod mounts the node's
  `/var/mnt/local-path-provisioner` as a `hostPath`. The cluster's default Pod
  Security level is `baseline`, which forbids `hostPath` volumes, so the
  namespace is labelled `privileged`. Talos's guide does the same.

## Capacity

The upstream README is explicit: capacity limits are not supported, and a
requested size is ignored. The StorageClass still reports
`allowVolumeExpansion: true`, but because nothing enforces the size, resizing a
claim has no effect.

The real limit is free space on `EPHEMERAL`. That space is shared with etcd,
container images, logs and the kubelet, because a `directory` user volume has no
filesystem of its own. A runaway volume can fill `/var` and take etcd down with
it. Longhorn had the same exposure, because its data also lives on `EPHEMERAL`
(`/var/lib/longhorn`).

Check free space on `EPHEMERAL`, and how much of it the volumes use:

```bash
# Node filesystem (EPHEMERAL) capacity / used / available, in GB
kubectl get --raw /api/v1/nodes/homelab-node-01/proxy/stats/summary \
  | jq '.node.fs | {capacityGB: (.capacityBytes/1e9), usedGB: (.usedBytes/1e9), availableGB: (.availableBytes/1e9)}'

# Disk usage per namespace under the provisioner's root
talosctl usage --nodes 192.168.0.10 --humanize --depth 1 /var/mnt/local-path-provisioner
```

The `talosctl` commands in this document use the node's LAN address, as
configured in `talosconfig`. Outside the LAN, pass the tailnet IP instead; see
[talos-operations.md](talos-operations.md#prerequisites).

## Operations

### Apply the Talos user volume

The user volume is declared in `infra/talos/talconfig.yaml` and, like every
Talos change, is applied by hand after the change merges. Apply it before any
claim uses the `local-path` class. A claim created earlier does not fail: the
helper pod mounts the volume's parent directory as a `DirectoryOrCreate`
`hostPath`, so it provisions into a plain directory it creates under
`/var/mnt/local-path-provisioner` on `EPHEMERAL`, outside the user volume Talos
manages. Adding the document does not require a reboot.

```bash
cd infra/talos
talhelper genconfig --no-gitignore
talosctl apply-config --nodes 192.168.0.10 \
  --file clusterconfig/homelab-cluster-homelab-node-01.yaml

# Expect phase `ready`, type `directory`
talosctl get volumestatus u-local-path-provisioner --nodes 192.168.0.10
talosctl get mountstatus u-local-path-provisioner --nodes 192.168.0.10
```

The volume has to survive a reboot. At the next planned reboot of the node (on a
single node this takes every workload down, so schedule it), re-run the two
`get` commands above, and confirm that the data written in the end-to-end check
below is still there. Keep the `lpp-test` claim until that reboot has been
checked.

### Verify provisioning end to end

A claim is only bound once a pod uses it (`WaitForFirstConsumer`), so the check
needs both:

```bash
kubectl create namespace lpp-test
kubectl apply -n lpp-test -f - <<'EOF'
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: test
spec:
  storageClassName: local-path
  accessModes: [ReadWriteOnce]
  resources:
    requests:
      storage: 1Gi
---
apiVersion: v1
kind: Pod
metadata:
  name: test
spec:
  containers:
    - name: test
      image: docker.io/library/busybox:1.37.0
      command: [sh, -c, 'test -f /data/marker || date > /data/marker; cat /data/marker; sleep 3600']
      volumeMounts:
        - name: data
          mountPath: /data
  volumes:
    - name: data
      persistentVolumeClaim:
        claimName: test
EOF

kubectl wait -n lpp-test --for=condition=Ready pod/test --timeout=120s
kubectl logs -n lpp-test test        # the timestamp just written

# Restart the pod; the same timestamp must come back
kubectl delete -n lpp-test pod test
# ...re-apply only the Pod from above, wait, then:
kubectl logs -n lpp-test test

# The directory is named after the namespace and claim
talosctl ls --nodes 192.168.0.10 /var/mnt/local-path-provisioner/lpp-test/test
```

Also confirm that Flux applied the component without PodSecurity violations.
The helper pod is created on demand, so a missing `privileged` label would first
surface when a claim is provisioned:

```bash
flux get kustomizations infrastructure
flux get helmreleases -n local-path-storage
# A rejected pod shows up as a FailedCreate warning here
kubectl get events -n local-path-storage --field-selector type=Warning
# Other namespaces log `restricted` warnings routinely; only local-path ones matter
kubectl logs -n flux-system deploy/kustomize-controller --since=1h \
  | grep -i podsecurity | grep -i local-path
```

Clean up once the reboot check above is done too. Because the class uses `Retain`, deleting the claim leaves
the PV and its directory behind, and both need the steps below.

### Release a retained volume

With `Retain`, deleting a claim moves its PV to `Released` and leaves its data on
disk. Nothing reclaims either one on its own.

```bash
# Find released local-path volumes and the directory each one holds
kubectl get pv -o custom-columns=NAME:.metadata.name,STATUS:.status.phase,NAMESPACE:.spec.claimRef.namespace,CLAIM:.spec.claimRef.name,CLASS:.spec.storageClassName,PATH:.spec.hostPath.path \
  | awk 'NR==1 || ($2=="Released" && $5=="local-path")'
```

Once you are sure the data is not needed (or it is in R2), delete the PV with
`kubectl delete pv <name>`, then remove its directory. The
directory is the `PATH` column above, and it is not removed by deleting the PV.
Talos has no shell, so remove it from a short-lived pod in this namespace:

```bash
kubectl run -n local-path-storage rm-volume --rm -it --restart=Never \
  --image=docker.io/library/busybox:1.37.0 \
  --overrides='{"spec":{"volumes":[{"name":"root","hostPath":{"path":"/var/mnt/local-path-provisioner"}}],"containers":[{"name":"rm-volume","image":"docker.io/library/busybox:1.37.0","stdin":true,"tty":true,"command":["sh"],"volumeMounts":[{"name":"root","mountPath":"/root-lpp"}]}]}}'
# inside: rm -rf /root-lpp/<namespace>/<claim>/<pv-name>
```

### Change the reclaim policy (after the migration)

`Retain` is temporary. A StorageClass's `reclaimPolicy`, `parameters` (which
include `pathPattern`), `provisioner` and `volumeBindingMode` are immutable in
the Kubernetes API, so changing any of them in `helmrelease.yaml` cannot be
applied as an in-place update. Plan it as its own change that replaces the
StorageClass.

A PV copies its reclaim policy from the StorageClass when it is created, so
replacing the class does not touch existing volumes. To change an existing PV,
patch it:

```bash
kubectl patch pv <name> -p '{"spec":{"persistentVolumeReclaimPolicy":"Delete"}}'
```

### Upgrades

Renovate's flux manager tracks `ref.tag` in `gitrepository.yaml` through the
`github-tags` datasource, and proposes bumps with the grouped `Helm charts` PR.
The tag itself is the pin. `pinDigests` is turned off for this dependency in
`renovate.json5`, because a tag ref has no field for a commit digest.
The provisioner image comes from the chart's defaults, so it moves with the tag.
The helper image lives in the HelmRelease values and arrives with the grouped
`container images` PR. See [renovate.md](renovate.md).

## Sources

Read 2026-09-23:

- [Talos v1.14 — User Volumes](https://docs.siderolabs.com/talos/v1.14/configure-your-talos-cluster/storage-and-disk-management/disk-management/user/) — `volumeType: directory`, mount point and kubelet propagation
- [Talos v1.14 — System Volumes](https://docs.siderolabs.com/talos/v1.14/configure-your-talos-cluster/storage-and-disk-management/disk-management/system/) — `VolumeConfig` applies only before a volume is provisioned
- [Talos — Local Storage](https://docs.siderolabs.com/kubernetes-guides/csi/local-storage) — local-path-provisioner on a user volume, privileged namespace
- [rancher/local-path-provisioner README](https://github.com/rancher/local-path-provisioner) — capacity limit, `pathPattern`
- [Flux — HelmChart `version` and `reconcileStrategy`](https://fluxcd.io/flux/components/source/helmcharts/) and [GitRepository `sparseCheckout`](https://fluxcd.io/flux/components/source/gitrepositories/)
- [Renovate — flux manager](https://docs.renovatebot.com/modules/manager/flux/) — `GitRepository` `tag` support
