# Lessons learned

Cross-incident operational wisdom for this cluster. Each entry distills a
generalizable lesson out of a specific event so future-self can apply it
without re-reading the entire incident report. New incidents should add (or
reinforce) entries here, not just file their own report.

Each entry is structured as:

- **Lesson** — the rule in one sentence.
- **Why** — what made it expensive to learn, with the concrete moment that
  taught it.
- **How to apply** — what to do differently next time.
- **Source** — which incident or PR(s) produced this lesson.

## Meta lessons (apply to any multi-component operation)

### When multiple things break at once, decompose before fixing

- **Lesson**: "Things broke at the same time" does not mean "things broke for
  the same reason." Build a list of distinct failures with distinct
  hypothesised triggers before jumping to a single explanation.
- **Why**: During the 2026-05-04 outage the reflex narrative was "Talos
  upgrade broke everything" and the first 30 minutes went toward "should we
  rollback Talos?". The actual structure was three independent failures —
  pod egress (Talos 1.13 trigger), Flux liveness (Cilium 1.19 trigger), and
  GitOps deadlock (consequence of the previous two) — and Talos rollback
  would only have fixed one of them while introducing its own new
  rollback-related issues.
- **How to apply**: As the very first investigation step, write down the
  symptoms as discrete bullets. Tag each with the most recent change that
  could have caused it. If two symptoms tag to different changes, treat them
  as separate problems with their own bisection until evidence forces you
  to merge them again.
- **Source**: [docs/incidents/2026-05-04-talos-1.13-cilium-1.19-tailscale.md](incidents/2026-05-04-talos-1.13-cilium-1.19-tailscale.md).

### When a config change has no observable effect, the change probably is not effective

- **Lesson**: If a setting "should" change behavior and behavior does not
  change, the most likely explanation is that the setting is silently
  ignored. Verify it propagated to where it actually runs before debugging
  anything else.
- **Why**: We hit this trap twice in the same incident:
  - `installIptablesRules: false` in the Cilium HelmRelease — `helm get
    values` showed it was being passed, but the chart in 1.19.3 has no such
    field, so it was silently dropped. The agent kept doing exactly what it
    was doing before. Two cycles of "wait and see if it stops" were wasted
    on this.
  - `TS_NETFILTER_MODE=iptables` in the Tailscale extension config — the
    extension service config showed the variable was set, but the variable
    name was wrong. Tailscale reads `TS_DEBUG_FIREWALL_MODE`. The applied
    config did literally nothing.
- **How to apply**: After applying a config change, **before** observing the
  intended behavior, verify the change reached the runtime. Concretely:
  - For Helm values: read the rendered ConfigMap / DaemonSet / Deployment,
    not `helm get values`. If you expect a key like `install-iptables-rules:
    "false"`, grep the actual `cilium-config` ConfigMap for it.
  - For container env vars: the container's `/proc/<pid>/environ` is the
    authoritative answer. The pod spec or service config may have keys that
    the binary does not actually read.
  - For "set X and the system should do Y" claims, find the upstream source
    code that reads X. If you cannot find code that reads X, X is probably
    not the right knob.
- **Source**: PRs #206, #208, #209, #210, #211 of the 2026-05-04 incident.

### Before calling a slow operation stuck, find its progress counter

- **Lesson**: "No output for N minutes" is not evidence of a stall. Locate the
  number that actually advances — bytes on disk, rows processed, offset
  committed — and sample it twice before forming any theory about *why* it is
  slow.
- **Why**: A `talosctl upgrade` hung three times at Talos's 20-minute pull cap
  with no client-side output at all. The pull was treated as stalled, and three
  separate theories (host network path, IPv6, MTU) were investigated on that
  assumption. The ground truth was one command away the whole time:
  containerd's ingest file was growing the entire time, 0 → 73 → 87 → 95 MiB
  across attempts. Sampling it first would have reframed the question from "why
  is it stuck?" to "why is it slow?" — a different and much smaller search.
  Relatedly, the first attempt was reported as *succeeding* because the command
  was piped (`talosctl upgrade ... | tail`), so `$?` was `tail`'s exit status,
  not `talosctl`'s.
- **How to apply**: For any long-running operation, ask "what number proves
  this is advancing?" before asking anything else, and sample it at two points
  in time. For Talos image pulls that number is the containerd ingest size (see
  *Upgrade Stuck* in `docs/talos-operations.md`). Never pipe a command whose
  exit status you intend to read — redirect to a file and capture `$?` on its
  own line.
- **Source**: #315 / #317, 2026-09-21 Talos v1.14.1 upgrade attempt.

### Benchmark the identical work, not a cheaper proxy of it

- **Lesson**: A benchmark only supports a conclusion if it does the *same work*
  as the slow path. Different input, different offset, or different object is a
  different measurement, however similar the command looks.
- **Why**: While diagnosing the slow pull above, throughput was "measured" with
  range requests for the first 10 MiB of the blob, from a pod (14.2 MB/s) and
  from a laptop (7.5 MB/s) — while the node was crawling through bytes 80 MiB
  and beyond at ~0.06 MB/s. The conclusion drawn was "the host pull path is
  100x slower than the pod path", and it was wrong: the CDN served the object's
  *head* fast and its *tail* slowly, so head-range benchmarks said nothing
  about the node's situation. Measuring the same tail range from the laptop
  reproduced the node's slowness exactly (0.10 MB/s), which relocated the
  problem from this cluster to the upstream origin in one command.
- **How to apply**: Before comparing A to B, write down what differs between
  them; if anything does, fix that first. For partial transfers, benchmark the
  same byte range. When a measurement exonerates your own infrastructure,
  re-run it in the failing condition before believing it.
- **Source**: #315 / #317, 2026-09-21 Talos v1.14.1 upgrade attempt.
### A key is its name plus its bytes; matching bytes is not a matching key

- **Lesson**: When you verify that a cryptographic key survived a migration,
  verify every field the decryption path keys off — not just the secret
  material. For Kubernetes secretbox/AES-CBC providers that includes the key
  **name**, which is written into every ciphertext and must match on read.
- **Why**: #317 migrated `talconfig.yaml` to Talos 1.14's multi-document form.
  The review explicitly checked the etcd encryption key and recorded it as
  "byte-identical" — the sha256 of the secret matched on both sides, and it
  did. What went unchecked was that Talos names the secretbox key `key2` while
  the generated `KubeEtcdEncryptionConfig` names the very same bytes `key1`.
  Kubernetes stores the key name in the ciphertext prefix, so the apiserver
  could not decrypt a single existing Secret. It sat at 0/1 for ~20 minutes
  and took Flux, the Tailscale ingress and kube-controller-manager down with
  it — from a config change whose diff looked like a rename.
- **How to apply**: For any credential or key that moves between config
  formats, enumerate the fields the *consumer* matches on and diff all of them.
  "The secret is the same" is a claim about one field. When the consumer is
  Kubernetes encryption-at-rest, the fields are provider type, key name and key
  bytes, in that order of subtlety.
- **Source**: #315 / #317, 2026-09-22 apiserver outage.

### When CNI logs disagree with intuition, read the kernel state directly

- **Lesson**: For datapath / netfilter / eBPF problems, the kernel's actual
  state is the only authoritative source. Do not stop at the CNI agent's
  error message — that message tells you the **symptom**, not the **cause**
  (it does not know who else is writing to the kernel).
- **Why**: Cilium's "table 'nat' is incompatible, use 'nft' tool" line told
  us the iptables manager was failing. It did not, and could not, tell us
  that **Tailscale** was the writer of the offending nat-table chain. We
  only learned that by scheduling a privileged debug pod with hostNetwork
  and running `nft list ruleset`, which revealed the `ts-postrouting` chain
  with no `# Warning: managed by iptables-nft` comment. Without that step,
  the next move would have been "Cilium has a bug in 1.19.3" — which would
  have led nowhere.
- **How to apply**: Keep the [`nft-inspector` Job
  YAML](incidents/2026-05-04-talos-1.13-cilium-1.19-tailscale.md#step-3--read-the-actual-kernel-netfilter-state)
  ready to apply for any CNI / netfilter weirdness. The image
  `nicolaka/netshoot` already includes `nft`, `iptables-nft`, `tcpdump`,
  `ss`, etc. — no `apk add` step that can hang on egress. Always read the
  kernel state before forming a hypothesis about which component is at
  fault. Look for **who else** is writing to the structure that is
  misbehaving — markers like the iptables-nft warning comment are the most
  reliable identifiers.
- **Source**: 2026-05-04 incident, "Step 3 — read the actual kernel
  netfilter state".

### Reproduce the failure with a different binary before blaming a binary

- **Lesson**: When a binary reports an error, try to reproduce the same
  error from another, independent binary that should hit the same kernel
  state. If both fail, the kernel state is the problem, not the binary. If
  only one fails, the binary is the problem.
- **Why**: Cilium's iptables-nft hit "table 'nat' is incompatible". The
  natural next thought was "Cilium 1.19.3 ships a buggy iptables-nft" — but
  running `iptables-nft -t nat -S` from the netshoot debug pod (different
  iptables-nft binary, same kernel) hit the **exact same error**, which
  pinned the problem to the kernel side. That cross-check eliminated an
  entire branch of the search tree (Cilium upgrade speculation) in seconds.
- **How to apply**: Always carry a second tool that can probe the same
  resource. `iptables-nft` vs `iptables-legacy` for netfilter, multiple CNI
  agents in a chain for CNI behavior, two API servers behind the same
  endpoint, etc. Symmetry of failure across independent tools is the
  fastest way to localize a problem.
- **Source**: 2026-05-04 incident.

### When GitOps deadlocks, imperative break-glass is OK; the second half is non-negotiable

- **Lesson**: If Flux cannot apply the very change that would fix Flux,
  manual `helm upgrade` / `kubectl apply` is the correct break-glass — but
  only if you immediately commit the equivalent change to git so the
  reconciliation lands on the same end-state.
- **Why**: During the 2026-05-04 outage, source-controller and helm-controller
  were in CrashLoopBackOff because Cilium could not provide working
  pod-to-external SNAT, which Flux needed to fetch the Cilium chart that
  would fix the SNAT. The classic chicken-and-egg. The break-glass was
  `helm upgrade cilium ... --set bpf.masquerade=true`. If we had stopped
  there, the next Flux reconcile would have unset `bpf.masquerade=true`
  and the cluster would have re-broken on schedule. PR #204 committing the
  same value to `helmrelease.yaml` was the second half that made the fix
  durable.
- **How to apply**: Treat any imperative cluster mutation as half a fix.
  Open the matching git PR before walking away from the keyboard. If you
  cannot push the matching PR right now (e.g., remote is unreachable),
  leave a note and a TODO; do not consider the incident closed.
- **Source**: 2026-05-04 incident, PRs #204, #205, #208, #211.

## Specific operational lessons

### `talosctl ... post check passed` is not a workload-health signal

- **Lesson**: A successful Talos OS upgrade only tells you that Talos
  itself is healthy. It does not tell you that the CNI re-attached cleanly,
  pods are Running, or external traffic flows.
- **How to apply**: After every Talos OS upgrade, run
  `kubectl get pods -A | awk '!/Running|Completed/'` and expect zero rows.
  If the sweep is non-empty, run the recovery procedure documented in
  [docs/talos-operations.md](talos-operations.md) (Cilium DaemonSet
  rollout-restart first, then look at the agent log for incompatibilities
  introduced by the new kernel).
- **Source**: 2026-05-04 incident; codified in
  `docs/talos-operations.md` post-upgrade verification subsection.

### When applying a Talos `ExtensionServiceConfig` change, the kernel may not be reset

- **Lesson**: `talosctl apply-config` for an extension service restarts
  the extension's containerd task but **does not reset kernel-level state**
  the extension previously wrote (firewall rules, sysctls touched without
  a restore handler, etc.).
- **Why**: When changing the Tailscale extension's firewall mode, the old
  `ts-postrouting` chain in nft remained even after `apply-config`. Mode
  switches that depend on the old rules being absent require a node reboot
  (or a manual `nft delete chain` if you are very confident in what you
  are doing).
- **How to apply**: For any extension config change that affects
  firewall / routing / kernel state, plan for a node reboot in the same
  maintenance window. Single-node clusters incur ~2 minutes of full
  cluster downtime; this is the cheaper-than-debugging-stale-state
  trade-off.
- **Source**: 2026-05-04 incident, PRs #209 / #211.

### `autoApprovers` only skips the manual step for the tag that actually requests it

- **Lesson**: A Tailscale ACL `autoApprovers.services` entry only auto-approves
  a service for the **tag that requests the approval**, not any tag related
  to the feature. For ProxyGroup-based Ingress, the device that advertises
  each new VIP Service is the proxy replica (tagged `tag:k8s` in this
  cluster's ACL), not the operator control-plane pod (tagged
  `tag:k8s-operator`). An `autoApprovers` list containing the wrong tag looks
  correct at a glance — the key exists, the feature reads as "configured" —
  but has no effect.
- **Why**: While deploying Trilium (issue #249/#250), its new
  `https://trilium.<tailnet>.ts.net` Ingress sat unreachable for ~40 minutes:
  `kubectl describe ingress trilium` showed an empty `status.loadBalancer`
  and the TLS secret (`trilium.<tailnet>.ts.net` in the `tailscale`
  namespace) had 0-byte `tls.crt`/`tls.key`, while a known-working app's
  Ingress had both populated. `kubectl rollout restart deployment/operator
  -n tailscale` did nothing, because the operator's desired state hadn't
  changed — it was waiting on an external approval it had already requested
  once. `tailscale status --json` confirmed `ingress-proxies-0` carries
  `tag:k8s`, but the tailnet's ACL had `"autoApprovers": {"services":
  {"svc:*": ["tag:k8s-operator"]}}` — the wrong tag — so every new Ingress
  required manually clicking "Approve" in the Tailscale Admin Console under
  **Services** before its cert could be issued and it became reachable.
- **How to apply**: When a new Tailscale Ingress hostname doesn't become
  reachable after Flux has reconciled it, check in this order before
  assuming a manifest or operator bug:
  1. `kubectl get ingress <name> -n <ns> -o yaml` — `status.loadBalancer: {}`
     (vs. a populated `hostname`/`ports` block on a working app) means the
     operator is still waiting on something external.
  2. `kubectl get secret <hostname>.<tailnet>.ts.net -n tailscale -o
     jsonpath='{.data.tls\.crt}' | wc -c` — `0` means no certificate has
     been issued yet.
  3. `kubectl logs -n tailscale ingress-proxies-0 --since=10m | grep -i
     cert` — a `starting SetDNS call` line with no following `got cert`
     means the ACME flow is stalled, almost always on a pending approval.
  4. Check the Tailscale Admin Console → **Services** for a "Needs
     approval" entry. If found, verify `autoApprovers.services` in the ACL
     policy lists the tag actually assigned to the advertising device
     (`tailscale status --json`, the `Tags` field for that peer) — not just
     any tag that sounds related.
- **Source**: Issue #249 / PR #250, 2026-07-05.

### A Talos image pull is capped at 20 minutes, but containerd resumes it

- **Lesson**: `talosctl upgrade` aborts the installer pull at exactly 1200 s
  (`ImageService/Pull ... 20m0.003s ... timeout`), and the node is untouched
  when it does — no staging, no reboot, nothing to roll back. containerd keeps
  the partial blob, so a repeated attempt continues where the last one stopped
  instead of starting over.
- **Why**: Three consecutive 20-minute failures looked like three total
  failures. They were not: the ingest grew monotonically across them
  (73 → 87 → 95 MiB of 157.9 MiB). Knowing the failure is resumable changes the
  decision — "retry costs nothing and accumulates" is a different calculus from
  "retry burns 20 minutes for nothing".
- **How to apply**: On a pull timeout, check the ingest size before deciding
  anything (recipe in `docs/talos-operations.md`, *Upgrade Stuck*). If it is
  advancing, the options are wait or retry; if it is flat, the problem is
  elsewhere. Never conclude the cluster is damaged by a pull timeout — it fails
  before the node is touched.
- **Source**: #315 / #317, 2026-09-21 Talos v1.14.1 upgrade attempt.

### Confirm your own Tailscale is up before trusting the tailnet ingress sweep

- **Lesson**: The post-upgrade verification sweep probes six
  `*.tail19032f.ts.net` hosts. If the *operator's* Tailscale client is off,
  all six fail identically — which is indistinguishable from the datapath
  failure the sweep exists to catch.
- **Why**: During the 2026-09-21 attempt the operator's Tailscale happened to
  be off while the cluster was being checked. The sweep had not been reached
  yet, so nothing was misdiagnosed — but had the upgrade proceeded, six
  simultaneous failures would have read as exactly the 2026-05-04 symptom
  (services unreachable over the tailnet) and sent the investigation into a
  cluster that was in fact fine.
- **How to apply**: Run `tailscale status` on the machine doing the checking
  before the sweep, and treat a whole-sweep failure as "check the client first"
  rather than "the datapath broke". Partial failure (some hosts up, some down)
  is the signal that actually implicates the cluster. Note this cuts the other
  way too: `talosctl`/`kubectl` against `192.168.0.10` go over the LAN and keep
  working with Tailscale off, so a working `kubectl` does not prove the tailnet
  path is healthy.
- **Source**: #315 / #317, 2026-09-21 Talos v1.14.1 upgrade attempt.
### Talos names the secretbox key `key2`, but its generated 1.14 document says `key1`

- **Lesson**: Talos's runtime builds the apiserver encryption config with the
  secretbox key named **`key2`** (`key1` is reserved for AES-CBC) — see
  `k8stemplates/apiserver.go`, identical in v1.13.9 and v1.14.1. The
  `KubeEtcdEncryptionConfig` document that Talos *generates* for 1.14 names the
  same key **`key1`**. Applying the generated document to a cluster whose
  Secrets predate it makes every Secret undecryptable.
- **Why**: The symptom does not name the cause. `kubectl` keeps working
  (it talks to `:6443` directly), so the cluster looks up; what you see is
  kube-apiserver stuck 0/1 with `readyz` reporting only
  `[-]informer-sync failed`, and everything that reaches the API through the
  `10.96.0.1` service VIP failing with `connection refused`. The real message is
  buried in the apiserver log: `unable to transform key
  "/registry/secrets/...": no matching key was found for the provided Secretbox
  transformer`. Reading that line is what turns a 20-minute outage into a
  five-minute one.
- **How to apply**: Before applying a regenerated machine config on a cluster
  that has been through the 1.13 → 1.14 boundary, check the key name:
  `talosctl read /system/secrets/kubernetes/kube-apiserver/encryptionconfig.yaml`
  against the `KubeEtcdEncryptionConfig` document in the generated file. If they
  disagree, do not apply — rotate first (procedure in
  `docs/talos-operations.md`). Note the trap is one-directional: it only bites
  clusters carrying pre-1.14 data, so a fresh cluster never sees it.
- **Source**: #315 / #317, 2026-09-22 apiserver outage.

### Talos 1.14's generated encryption document cannot be overridden by a patch

- **Lesson**: There is no declarative way to change the generated
  `KubeEtcdEncryptionConfig`. All four patch mechanisms fail, so a key-name
  mismatch must be fixed in the *data* (rotate the Secrets), not in
  `talconfig.yaml`.
- **Why**: Each failure mode is non-obvious and two of them fail silently
  enough to look like success. Measured on talhelper v3.1.17 / Talos v1.14.1:

  | Attempt | Result |
  |---|---|
  | `$patch: delete` the document | `etcd encryption config is required for control plane machines` |
  | Strategic merge of just the key name | **Appends a second `resources` entry**; Kubernetes uses the first match, so the old name still wins |
  | `$patch: replace` on `config` | Not honoured — the literal `$patch: replace` key is emitted into the generated config |
  | JSON6902 patch | `JSON6902 patches are not supported for multi-document machine configuration` |

- **How to apply**: Do not spend time trying to patch it. Rotate the Secrets to
  the name the generator uses, after which `talconfig.yaml` needs no encryption
  stanza at all and the generated config is correct as-is. That end state is
  also why this repository carries no encryption patch today.
- **Source**: #315 / #317, 2026-09-22 apiserver outage.

### Cilium 1.19+ enforces NetworkPolicy strictly enough to block kubelet probes

- **Lesson**: Any `NetworkPolicy` that selects pods with
  `from: namespaceSelector: {}` or `from: podSelector: {}` and `policyTypes:
  [Ingress]` blocks kubelet liveness/readiness probes under Cilium 1.19+
  because kubelet sources from the host network and matches neither
  selector.
- **How to apply**: When auditing `NetworkPolicy` against this cluster,
  ensure every Ingress-typed policy that selects pods with health probes
  either includes the host network in its `from` clause (e.g., `ipBlock`
  for the node CIDR) or excludes the relevant probe ports from its scope.
  The default Flux `flux-system` bootstrap policies fail this — see
  PRs #204 / #205 for the corrective change.
- **Source**: 2026-05-04 incident, PRs #204 / #205.

### Switching a Deployment to `Recreate` needs the defaulted `rollingUpdate` block cleared

- **Lesson**: Changing `spec.strategy.type` from `RollingUpdate` to
  `Recreate` in Git is not enough. The API server has already defaulted
  `spec.strategy.rollingUpdate` (`maxSurge: 25%`, `maxUnavailable: 25%`) onto
  the live object, server-side apply merges the new `type` on top of it
  rather than dropping it, and the API server then rejects the result:
  `spec.strategy.rollingUpdate: Forbidden: may not be specified when strategy
  type is 'Recreate'`. Clear the field imperatively once, then reconcile.
- **Why**: On 2026-09-22 this took down the reconciliation of the entire
  `apps` Kustomization — not just Home Assistant — because one invalid
  object fails the whole dry-run. Flux reported `Ready=False` with the
  message above while every other app in `k8s/apps` silently stopped being
  reconciled. The merged PR looked fine; only the Kustomization status said
  otherwise, and nothing in the cluster degraded, so there was no symptom to
  notice from the outside.
- **How to apply**: When a PR switches a Deployment to `Recreate`, expect to
  follow the merge with:

  ```bash
  kubectl -n <ns> patch deploy <name> --type=merge \
    -p '{"spec":{"strategy":{"type":"Recreate","rollingUpdate":null}}}'
  flux reconcile kustomization <ks> --with-source
  ```

  This is the legitimate half of the break-glass rule above — Git already
  holds the desired state, and the patch only removes a defaulted field that
  was preventing Git from being applied, so there is no second commit owed.
  After any such merge, check `flux get kustomizations` rather than assuming
  a green PR means a reconciled cluster. The same applies to Helm-managed
  Deployments: Helm's three-way merge will not remove a field that was never
  in its own previous manifest either.
- **Source**: PRs #324 (home-assistant), #312, and the follow-up switching
  audiobookshelf and immich-server.

### Single-replica apps on RWO volumes need `Recreate`, and most will not tell you

- **Lesson**: A `replicas: 1` Deployment with no `strategy` gets
  `RollingUpdate` with `maxSurge: 25%`, which rounds **up to 1**. Every
  rollout therefore runs two pods at once, both mounting the same RWO volume
  on the same node, until the new one is Ready.
- **Why**: Home Assistant 2026 takes an exclusive `fcntl.flock` on its config
  directory and refuses to start, which surfaced as a CrashLoopBackOff and a
  rollout that could never complete — the old pod holds the lock and is never
  retired because the new pod never turns Ready. That was the *lucky* case:
  it failed loudly. Audiobookshelf, on the same pattern, had been running two
  processes against one SQLite database for a few seconds on every rollout
  for months without complaint.
- **How to apply**: Any Deployment here with `replicas: 1` and a
  `persistentVolumeClaim` volume should declare `strategy: Recreate`. For
  bjw-s-common-based charts (immich) the value is
  `controllers.<name>.strategy: Recreate`. Accept the brief downtime: at one
  replica the rollout is not highly available anyway, it only looked like it.
- **Source**: PRs #312, #324, and the follow-up switching audiobookshelf and
  immich-server.

## The single most valuable takeaway

If you read nothing else, read this:

> **Verify the change took effect before observing the intended behavior.**

We burned roughly an hour across the 2026-05-04 incident debugging "why is
the symptom unchanged?" when in fact the proposed fix had been silently
dropped (Cilium chart, `installIptablesRules`) or read from a non-existent
env var (Tailscale, `TS_NETFILTER_MODE`). Both were avoidable with a single
verification step between *apply* and *observe*: read the rendered
ConfigMap / DaemonSet / `/proc/<pid>/environ`, confirm the exact key/value
is present where it actually executes, **then** decide whether the new
behavior matches expectation.

A productive habit:

1. Apply.
2. Verify the config reached the runtime (rendered manifest, not the input).
3. Observe behavior.
4. Decide whether (3) matches expectation.

If (2) fails, do not run (3) and (4) — fix the config-propagation problem
first.
