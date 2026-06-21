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
