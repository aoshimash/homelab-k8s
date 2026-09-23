# Storage migration decision: Longhorn → local-path-provisioner + K8up

- **Date**: 2026-09-23
- **Status**: Accepted; implementation tracked by [#299](https://github.com/aoshimash/homelab-k8s/issues/299)
- **Supersedes**: `specs/002-longhorn-r2-backup/` (2026-01-02)

Statements about the cluster in the present tense describe it **as of
2026-09-23**, before any of the migration has landed. They are the starting
state this decision argues from, not a running description of the cluster.

Longhorn is being removed from this cluster. This record exists because the
reasoning cannot be recovered from the resulting code: a reader who finds
local-path-provisioner and K8up in place has no way to tell that Longhorn was
considered, measured and deliberately dropped, or that node-to-node replication
was rejected rather than forgotten. The record is written before the migration
rather than after it, while the measurements and the rejected options are still
first-hand.

## Decision

Replace Longhorn with:

| Concern | Before | After |
|---|---|---|
| Storage | Longhorn distributed block storage, `defaultReplicaCount: 1` | Talos **User Volume** + **rancher/local-path-provisioner** |
| Backup | Longhorn recurring jobs → Cloudflare R2 | **K8up** (restic) → Cloudflare R2 |
| Backup scope | every volume, implicitly | **opt-in**: only volumes explicitly marked |

The durability posture is deliberately unchanged in kind — one copy of the data
on local disk, daily backups to R2. What changes is the machinery that provides
it, and the maintenance that machinery costs.

Talos User Volumes *"allow to treat available disk space as a pool of allocatable
resource, which can be dynamically allocated to different applications"*, and a
user volume is *"automatically mounted under `/var/mnt/<user-volume-name>` path on
the node"* — which is exactly the root path local-path-provisioner needs. K8up is
*"Based on top of Restic, it can store backups in any S3-compatible storage"*, and
R2 is S3-compatible, so the existing backup target is reachable unchanged.

## Why Longhorn is being replaced

### The decisive reason is upgrade cadence, not resource usage

Longhorn does not support skipping minor versions. Its own upgrade
documentation for the version this repository pins states: *"We only support
upgrading to v1.12.1 from v1.11.x. For other versions, please upgrade to v1.11.x
first."* This repository had already written the rule down for itself, in
`k8s/infrastructure/longhorn/helmrelease.yaml`, in the form it took at the time
(`v1.7.3 → v1.8.x → v1.9.x → v1.10.x`) — that file goes away with the component,
and git keeps it.

Every other dependency here is pinned in Git and updated by Renovate as a
reviewable PR — that is the whole maintenance model of this repository (see
[docs/renovate.md](renovate.md)). Longhorn cannot participate in it: each bump is
a manual, staged, cluster-side upgrade path rather than a review-and-merge.
Paying that indefinitely to run a distributed storage system at replica 1 is not
a good trade.

This is the reason that survives every counter-argument below. The resource
numbers could be fixed without removing anything; the upgrade burden could not.

### Measurements (live cluster, 2026-09-21)

| Measurement | Value |
|---|---|
| Running pods in `longhorn-system` | 23 |
| DaemonSets | 7, of which **5 are stale `engine-image-*`** aged 261d / 260d / 234d / 91d / 22d — residue of the upgrade chain from v1.7.3 |
| CPU / memory actually in use | 116m / 1383Mi (13% of the node's in-use memory) |
| `instance-manager` CPU **requests** | **1914m — 53% of the cluster's entire CPU requests** |
| Data served | ~13.2Gi actual across 10 PVCs (206Gi provisioned), **no replication** |

The 1914m is not a misconfiguration; it is the documented default. Longhorn's
`Guaranteed Instance Manager CPU` defaults to `{"v1":"12","v2":"12"}`, documented
as *"Percentage of the total allocatable CPU resources on each node to reserve
for each instance manager pod. For example, a value of `10` means 10% of the
total CPU on a node will be allocated to each instance manager pod on that
node."* At the default 12: 15950m allocatable × 0.12 = 1914m.

Replication is not in use and is not wanted. Longhorn's best practices page
recommends *"10 Gbps network bandwidth between nodes"* for optimal volume
performance: a cost that is not exercised by a single node at replica 1, but one
that would start to matter the moment a second node joined with replica ≥ 2.
Removing Longhorn removes that future constraint along with the present one.

### What changed after the measurement

Immich was removed from the cluster on 2026-09-22 (`31ce861`) — it held no data.
Its 100Gi PVC (~250MB actual) therefore leaves the figures above, and
`immich-library`, listed as in-scope for backup when #299 was written, is no
longer a volume that exists. The measurement table is kept as it was taken; this
note is the delta. Corrected for it, the numbers as of 2026-09-23 are **~106Gi
provisioned against ~13Gi actual**, and the backup set below totals **~11Gi**.

## Alternatives rejected

### Tune Longhorn in place

Lowering `Guaranteed Instance Manager CPU` would recover the 1914m immediately
with zero migration risk, and pruning the stale `engine-image-*` DaemonSets is
routine. This was the cheapest option on resources and was rejected anyway: it
fixes the numbers and leaves the upgrade constraint — the recurring cost —
entirely in place. A one-time migration cost is accepted here specifically to
remove a recurring one.

### VolSync as the backup layer

VolSync is restic-backed and would have worked. It was rejected on per-volume
boilerplate: its documentation states *"Each PVC will be backed up with a
separate ReplicationSource, and each should use its own separate restic-config
secret"*, and *"If backing up multiple PVCs to the same S3 bucket, the path
underneath the bucket must be unique for each PVC."* Every new volume would then
add a resource plus a SOPS-encrypted secret plus a unique bucket path. The goal
of this migration is less per-volume ceremony, not more.

### TopoLVM as the storage layer

TopoLVM would restore what local-path-provisioner gives up — real capacity
enforcement, snapshots and volume expansion. It was rejected as disproportionate
to ~13Gi of data on one node, and because of what it requires underneath: its
design puts an `lvmd` gRPC service on each node to manage LVM volumes, and *"A
volume group must be created on all nodes where TopoLVM will run."* Providing
LVM2 volume groups and an `lvmd` on a Talos host is not a documented Talos path
— unlike local-path-provisioner on a user volume, which Talos documents directly.
Taking on an undocumented host-level dependency to protect 13Gi is the wrong
trade.

## Accepted trade-offs

These are chosen, not overlooked.

- **No capacity enforcement.** local-path-provisioner's README is explicit: its
  "Cons" section lists *"No support for the volume capacity limit currently."*
  and, beneath it, *"The capacity limit will be ignored for now."* A PVC's
  `resources.requests.storage` becomes documentation rather than a limit, and
  volume expansion is meaningless rather than available. The over-provisioning
  this cluster carries (~106Gi requested against ~13Gi in use, post-Immich)
  simply stops being a concept; the real limit becomes free space on the user
  volume — a node-level concern rather than a per-PVC one.
- **No volume snapshots.** Recovery is from the R2 backup, not from a local
  point-in-time copy.
- **Data is lost on node failure.** Talos states it plainly: *"Local storage is
  not replicated, so in case of a machine failure contents of the local storage
  will be lost."* This is the same exposure as today's replica-1 Longhorn
  configuration — it is not a new risk, it is an existing one made explicit.
  Recovery is from R2.
- **The Longhorn UI goes away.** Volume and backup observability moves to K8up's
  Prometheus metrics through the existing Grafana Alloy → Grafana Cloud path
  (see [docs/grafana-alloy.md](grafana-alloy.md) and
  [docs/backup-alerting.md](backup-alerting.md)). Backup-failure alerting
  already exists for the Longhorn and CloudNativePG backups and travels the same
  path, so what is lost is the by-hand view, not the signal.

## Why backups are opt-in

Backups are opt-in — a volume is backed up only when explicitly marked. This is
a deliberate requirement, not a default that happened to be convenient.

The selection axis is a single question: **can this data be regenerated from
elsewhere?** If yes, it is not backed up.

| Volume | Backed up | Why |
|---|---|---|
| `audiobookshelf-config` | yes | Hand-built configuration |
| `audiobookshelf-metadata` | yes | Accumulated listening state, not re-derivable |
| `audiobookshelf-podcasts` | yes | Most feeds have ended, so re-fetching is not possible |
| `paperless-data` | yes | Document index and state |
| `paperless-media` | yes | Scanned originals — the archive itself |
| `home-assistant-config` | yes | Hand-built configuration and history |
| `vikunja-files` | yes | User-uploaded attachments |
| `postgres-cluster-1` (CNPG PGDATA) | **no** | Covered by CloudNativePG's own barman backups to R2 |
| `data-vikunja-postgresql-0` | **no** | Not migrated either — an orphan from the move to CloudNativePG (`state=detached`, `robustness=unknown`, and the `vikunja` namespace has no StatefulSet), to be deleted by [#305](https://github.com/aoshimash/homelab-k8s/issues/305) |

`immich-library` was in this list when #299 was written and is no longer, per the
delta noted above. The in-scope set totalled about 11.2Gi as measured on
2026-09-21, including Immich's ~250MB.

The PostgreSQL exclusion is the clearest argument for opt-in. Under Longhorn, as
it stands on 2026-09-23, PGDATA is not actually excluded. Nothing in this
repository selects recurring jobs per volume, so Longhorn's documented fallback
applies — every volume without a recurring-job label of its own lands in the
`default` group, and the `backup-daily` job backs all of them up. PGDATA is among
them: ~30 backups, ~41GB in R2 (`59a0f29`).

A file-level copy of a live PostgreSQL data directory is not a valid backup, so
that storage bought nothing. The exclusion this repository believed it had was
never real, and nothing surfaced that for months, because a volume was backed up
unless something excluded it and nothing reported that the exclusion had failed.
An opt-in model cannot fail that way: a volume is backed up because something
named it.

**Backup frequency stays daily.** Daily is the cadence already in place —
`specs/002` FR-010 for volumes, and CloudNativePG's own 18:00 UTC schedule for
the database. [#299](https://github.com/aoshimash/homelab-k8s/issues/299) records
that increasing it was considered and declined, and records no reason; none is
reconstructed here.

## What this supersedes

`specs/002-longhorn-r2-backup/` (2026-01-02) introduced Longhorn and its R2
backups. It stays exactly as it is — AGENTS.md marks `specs/` a read-only
historical archive, nothing has been committed anywhere under it since
2026-02-01, and `specs/002` itself has not been touched since 2026-01-02 — so
this record is the pointer that keeps it from reading as current.

Reversed by this decision:

| specs/002 | Status |
|---|---|
| FR-001 — Longhorn MUST be the cluster's persistent storage provider | **Reversed.** local-path-provisioner on a Talos user volume |
| FR-002 / FR-002a — single-node tuning, default replica count MUST be 1 | **Moot.** There is no replica count; there is one copy on local disk |
| FR-004 / FR-005 / FR-006 — backup and restore of *Longhorn volumes* to R2 | **Reversed in mechanism.** K8up/restic → R2. R2 as the target and a verified restore path both survive |
| FR-010 — automatic daily backups for volumes | **Reversed in scope.** Daily survives; *automatic for every volume* does not — backups are opt-in |
| FR-012 — retain backups for 30 days | **Not carried over.** Retention is set explicitly when K8up is configured ([#301](https://github.com/aoshimash/homelab-k8s/issues/301)). This record does not decide it, and 30 days does not carry over on its own |
| Clarification "Default replicas = 1" (session 2026-01-02) | **Superseded** by the decision that there is no node-to-node replication at all |

Not reversed, and still binding:

- FR-003 — dynamic provisioning, data retained across pod restarts.
- FR-011 — a scheduled backup over unchanged data should run without re-copying
  everything, in storage and in time. Carried by restic rather than by Longhorn's
  incremental backups: restic deduplicates, so *"no new data was added to the
  repository (since all data is already there)"*, and it skips rescanning with
  *"a change detection rule based on file metadata to determine whether a file is
  likely unchanged since a previous backup"* against a parent snapshot. Met by
  different means, not dropped — to be confirmed in practice when K8up is
  configured ([#301](https://github.com/aoshimash/homelab-k8s/issues/301)).
- FR-007 / FR-008 — backup credentials encrypted in Git with SOPS + age,
  decrypted by Flux at reconciliation. Unchanged; only the consumer changes.
- The intent behind FR-009 — operator-visible backup success/failure signals.
  With the Longhorn UI gone this is served by metrics and alerts rather than a
  dashboard.

## Sources

Primary sources, read 2026-09-23:

- [Longhorn — Upgrading Longhorn Manager (v1.12.1)](https://longhorn.io/docs/1.12.1/deploy/upgrade/longhorn-manager/) — supported upgrade paths
- [Longhorn — Settings reference (v1.12.1)](https://longhorn.io/docs/1.12.1/references/settings/#guaranteed-instance-manager-cpu) — `Guaranteed Instance Manager CPU` default
- [Longhorn — Best Practices (v1.12.1)](https://longhorn.io/docs/1.12.1/best-practices/) — 10 Gbps between nodes
- [Talos — Local Storage](https://docs.siderolabs.com/kubernetes-guides/csi/local-storage) — the replication warning, the `/var/mnt/<user-volume-name>` mount sentence, and local-path-provisioner rooted on a user volume. This guide is not version-scoped on the docs site; the User Volumes page below is, and is pinned to the version this cluster runs
- [Talos v1.14 — User Volumes](https://docs.siderolabs.com/talos/v1.14/configure-your-talos-cluster/storage-and-disk-management/disk-management/user/) — what user volumes are. v1.14 matches `talosVersion: v1.14.1` in `infra/talos/talconfig.yaml`
- [rancher/local-path-provisioner README](https://github.com/rancher/local-path-provisioner) — capacity limit
- [K8up documentation](https://docs.k8up.io/k8up/2.12/index.html) — restic, S3-compatible targets
- [restic — Backing up](https://restic.readthedocs.io/en/stable/040_backup.html) — deduplication, parent snapshot, metadata-based change detection
- [VolSync — restic usage](https://volsync.readthedocs.io/en/stable/usage/restic/index.html) — per-PVC ReplicationSource and secret
- [TopoLVM — design](https://github.com/topolvm/topolvm/blob/main/docs/design.md) and [getting started](https://github.com/topolvm/topolvm/blob/main/docs/getting-started.md) — lvmd, volume group prerequisite

Cluster measurements are from [#299](https://github.com/aoshimash/homelab-k8s/issues/299),
taken against the live cluster on 2026-09-21.
