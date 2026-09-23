# Node filesystem alerting (EPHEMERAL)

An alert that fires in Slack when a node's `EPHEMERAL` filesystem (`/var`) is
more than 70% used, and a second one that fires if the metric behind it
disappears. Implements
[#343](https://github.com/aoshimash/homelab-k8s/issues/343). Rules are in
`k8s/infrastructure/grafana-alloy/prometheusrule-node-filesystem.yaml`; they
reach Slack through the same path as the backup alerts (see
[backup-alerting.md](backup-alerting.md#architecture)).

## Why

`EPHEMERAL` is the node's one large filesystem (`/dev/nvme0n1p4`, 997GB, the
whole disk apart from Talos's system partitions). It holds etcd, container
images, the kubelet, logs, Longhorn's data (`/var/lib/longhorn`) and every
local-path volume (`/var/mnt/local-path-provisioner`). Nothing isolates one from
another.

local-path-provisioner does not enforce a volume's requested size (see
[local-path-provisioner.md](local-path-provisioner.md#capacity)), so one volume
can grow until `EPHEMERAL` is full. The kubelet does not step in until late, and
what it does then cannot free volume data:

- It starts image garbage collection at 85% used (`imageGCHighThresholdPercent:
  85`) and evicts pods below 10% free (`evictionHard` `nodefs.available: 10%`)
  or 15% free (`imagefs.available: 15%`). Images and the kubelet share this one
  filesystem, so eviction starts at about 85% used. These are the values the
  kubelet reported through `/configz` on 2026-09-23.
- Evicting a pod does not delete its PersistentVolume, so eviction frees no
  volume data.
- etcd is a Talos system service, not a pod. It is never evicted; it stays up
  until its writes fail.

The space is not scarce (113GB of 997GB, 11%, on 2026-09-23). The alert is for
growth nobody expected, such as a bug writing without bound. 70% leaves about
150GB before the kubelet acts.

## Alert rules

| Alert | Condition | Severity |
|-------|-----------|----------|
| `NodeEphemeralFilesystemUsageHigh` | `container_fs_usage_bytes{id="/"} / container_fs_limit_bytes{id="/"} > 0.7` for 15m | warning |
| `NodeFilesystemMetricsAbsent` | that expression returns nothing, for 1h | warning |

Both carry `component: node-filesystem`, so Alertmanager groups them apart from
the backup alerts. The usage alert clears once usage drops back to 70% or
below. The 15m hold keeps a burst, such as a large image pull, from firing it.

There is one level only. A critical level would sit where the kubelet's own
`DiskPressure` condition already turns true.

`NodeFilesystemMetricsAbsent` guards the usage alert: over a missing metric the
usage expression returns nothing, and an alert over nothing never fires. It
fires if the allow-list below is changed, the cAdvisor scrape breaks, or the
device label stops matching. It looks at all nodes together, so once there is a
second node it fires only when every node's series are gone; one node losing
its series goes unnoticed.

## Metrics used

The kubelet's cAdvisor endpoint (`/metrics/cadvisor`, Alloy's `cadvisor` scrape
job) exports `container_fs_limit_bytes` (size) and `container_fs_usage_bytes`
(used) for the node root container, `id="/"`, one series per filesystem it
sees, labelled by `device`. For `device="/dev/nvme0n1p4"` they match the
kubelet's own summary of `EPHEMERAL` (`.node.fs` in `stats/summary`).

The same names are exported for every container, and the root container also
reports 17 other mounts (tmpfs, overlay and bind mounts), some of them full by
design (e.g. a 128KiB mount at 100%). Alloy's `cadvisor_filter` therefore keeps
these two names only when `id="/"` and `device` is a `/dev/<name>` ending in a
digit (e.g. `/dev/nvme0n1p4`, `/dev/sda4`). That adds two series per node. On
homelab-node-01 the only such filesystem the kubelet sees is `EPHEMERAL`.

The rules do not name a device, so a second node's `EPHEMERAL` is covered
without a change. Any other matching device the kubelet comes to see, such as
a partition-type user volume or a loop mount, would be alerted on in the same
way, under the same `EPHEMERAL` summary. A read-only loop mount is always 100%
used, so if one appears, narrow the `device` part of the regex rather than
silencing the alert.

The alert's `instance` label is the scrape address, the node's InternalIP and
kubelet port (e.g. `100.119.230.112:10250`), not its name. `kubectl get nodes -o
wide` maps one to the other.

`kube_node_status_condition{condition="DiskPressure"}` (kube-state-metrics) is
not used: it turns true at the kubelet's eviction thresholds, which is what this
alert has to get ahead of.

## When it fires

1. Confirm the usage from the kubelet:

   ```bash
   kubectl get --raw /api/v1/nodes/homelab-node-01/proxy/stats/summary \
     | jq '.node.fs | {capacityGB: (.capacityBytes/1e9), usedGB: (.usedBytes/1e9), availableGB: (.availableBytes/1e9)}'
   ```

2. Find the directory that is growing. Use the node's tailnet IP from outside
   the LAN (see [talos-operations.md](talos-operations.md#prerequisites)):

   ```bash
   talosctl usage --nodes 192.168.0.10 --humanize --depth 1 /var/lib
   talosctl usage --nodes 192.168.0.10 --humanize --depth 2 /var/mnt/local-path-provisioner
   ```

   `talosctl usage` adds up file sizes, not the blocks they use, so sparse
   files count at their full size. Longhorn's replica files are sparse: on
   2026-09-23 `/var/lib/longhorn` showed 244GB while the whole filesystem used
   113GB. Compare directories with each other and over time, not with the
   filesystem total.

3. Typical causes and what to do:
   - **A local-path volume** (`/var/mnt/local-path-provisioner/<namespace>/<claim>/`):
     find the app writing without bound and fix or clean up at the app. A
     requested size does not limit it.
   - **`Released` local-path PVs** (the class uses `Retain`): their directories
     stay behind until removed by hand; see
     [local-path-provisioner.md](local-path-provisioner.md#release-a-retained-volume).
   - **Container images** (`/var/lib/containerd`): the kubelet only collects
     unused images from 85%. The kubelet's view of image usage is
     `kubectl get --raw /api/v1/nodes/homelab-node-01/proxy/stats/summary | jq '.node.runtime.imageFs'`.
   - **Logs** (`/var/log`) or **etcd** (`/var/lib/etcd`): unusual growth here
     points at a misbehaving component rather than a volume.

## Verifying

After the change merges and Flux reconciles:

1. The rules are in the ruler (`rules:read`, which the `homelab-alloy` token
   has, is enough):

   ```bash
   curl -s -u "<instance-id>:<token>" \
     "https://prometheus-prod-XX.grafana.net/api/prom/api/v1/rules?type=alert" \
     | jq '.data.groups[] | select(.name == "node-filesystem") | .rules[] | {name, state}'
   ```

   Both should be listed as `inactive`.

2. The usage expression evaluates against real data. In Grafana Cloud Explore:

   ```promql
   container_fs_usage_bytes{id="/"} / container_fs_limit_bytes{id="/"}
   ```

   It should return one series per node, `device="/dev/nvme0n1p4"` for
   homelab-node-01, at the current usage ratio (about 0.11 on 2026-09-23).
   Compare with the `stats/summary` command above.

3. Only the node root series were added:

   ```promql
   count by (id, device) (container_fs_usage_bytes)
   ```

   It should return `id="/"` with the block device only.

Waiting for the disk to reach 70% is not practical, so the firing path is not
exercised live; step 2 is the check that the rule sees real data.
