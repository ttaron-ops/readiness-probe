---
lesson: "01b.7"
title: "The network datapath: netfilter, conntrack, and NAT at scale"
module: "01b"
concept: "The network datapath: netfilter, conntrack, and NAT at scale"
status: not-started
est_time: "6h"
prev: "06-hugepages-thp-numa.md"
next: "08-ebpf.md"
artifacts: []
sources: 5
---

# 01b.7 · The network datapath: netfilter, conntrack, and NAT at scale

> **Concept.** Every NATed pod flow leaves a fingerprint in the kernel's connection-tracking table; when that table fills, a busy node drops packets silently — and eBPF datapaths exist largely to escape it.
>
> Module: [🐧 01b — Linux systems internals](../README.md) · Deliverable: [Anatomy of a Container](../practice/anatomy-of-a-container/README.md)

## Where this fits

Lesson 06 was about a *memory-side* shared resource under fleet load: hugepages/THP and NUMA locality, where the failure mode is TLB pressure and cross-socket latency that quietly steals throughput from a GPU or NIC. This lesson is the network-side sibling of that same pattern — a node-local kernel resource, shared across every pod on the box, that degrades invisibly until it doesn't. Here the resource is `nf_conntrack`, the kernel's connection-tracking table, and the mechanism is netfilter hooks + NAT. By the end you'll be able to read the hook order a packet actually takes, explain *why* NAT cannot exist without a conntrack entry per flow, recognize the exhaustion symptom sequence before it pages you, and articulate why Cilium's eBPF datapath sidesteps the problem instead of just tuning around it. That last point is the hinge into the next lesson: **08 — eBPF**, which unpacks the general-purpose mechanism (verifier, hooks, maps) that Cilium's socket-LB and BPF conntrack map are actually built from. You'll leave this lesson knowing *that* eBPF fixes the conntrack bottleneck; the next lesson explains *how* eBPF does anything at all.

## Why this matters

On a GPU cluster the network *is* the workload. Distributed training runs NCCL all-reduce across nodes; checkpointing and data ingestion pull terabytes from object storage; sidecars phone home to control planes. Two traffic shapes stress the kernel differently, and both route through the same chokepoint — **conntrack**:

- **NCCL / RDMA** ideally bypasses the host TCP/IP stack (RoCE/IB), but the control channels, health checks, and any TCP fallback still traverse netfilter.
- **Egress fan-out** — thousands of short-lived HTTPS connections from dataloaders scraping shards out of S3/GCS, metrics push, log shipping — creates a *new conntrack entry per flow*, and because pods egress via source-NAT (masquerade) to the node IP, the node's conntrack table is the shared resource that all pods contend for.

When `nf_conntrack_max` fills, the kernel logs `nf_conntrack: table full, dropping packet` and starts dropping *new* connections. The user-visible symptom is maddening: existing connections work, but new ones hang and time out — intermittently, under load, on your busiest nodes, exactly when a training job ramps its data pipeline. Knowing this mechanism is the difference between "the network is flaky" and "node conntrack is at 98%, here's the sysctl and here's why we should move to an eBPF datapath." That's a cost/observability differentiator: you catch it in a dashboard before it pages, and at a staff level you can make the case for *architectural* change instead of another sysctl bump.

## What's new here (calibration)

Per the module README's calibration: you already know shell/pipes/networking basics, Kubernetes Services in the abstract, and how to read `iptables -t nat -L` or `conntrack -L` output at a glance — none of that is re-taught here. What's genuinely new:

- The **mechanical coupling** between NAT and conntrack — why NAT is *implemented on top of* conntrack, not a separate feature, and why that means one conntrack entry per NATed flow is not a tunable, it's a structural fact.
- The **exhaustion symptom sequence** as a diagnostic fingerprint you pattern-match under pressure, not just a sysctl to raise after the fact.
- The **fairness/blast-radius framing**: a node-global table is a shared-resource problem, not merely a scaling limit — the angle a staff design review actually probes.
- The **mechanism** (not just the marketing line) behind "Cilium removes conntrack load" — socket-LB, BPF host routing, and Cilium's own eBPF conntrack map.

## Core concepts

### 1. The netfilter hooks and where conntrack sits

A forwarded packet (pod → outside world) hits hooks in order:

```
NIC → [PREROUTING] → routing → [FORWARD] → [POSTROUTING] → NIC
                                                 └─ SNAT/MASQUERADE happens here
locally-generated:  [OUTPUT] → routing → [POSTROUTING]
```

`nf_conntrack` registers at `PREROUTING` (and `OUTPUT`) to *look up or create* the flow's state early, and at `POSTROUTING` to *confirm* the entry and apply any NAT recorded for it. The NAT tables (`iptables -t nat`) only decide the translation once — on the first packet of a flow, in the `nat` table — and conntrack replays it for every subsequent packet. That's the efficiency trick and the liability: state per flow.

Inspect the machinery:

```
$ lsmod | grep nf_conntrack
$ sysctl net.netfilter.nf_conntrack_count net.netfilter.nf_conntrack_max
net.netfilter.nf_conntrack_count = 18342
net.netfilter.nf_conntrack_max = 262144
$ cat /proc/sys/net/netfilter/nf_conntrack_count   # same value, raw
$ conntrack -L | head        # live table dump (needs conntrack-tools)
tcp  6 431999 ESTABLISHED src=10.244.3.7 dst=52.94.1.2 sport=51042 dport=443 \
    src=52.94.1.2 dst=192.0.2.10 sport=443 dport=51042 [ASSURED] mark=0 use=1
```

Read that entry: the pod (`10.244.3.7`) opened `:443` to an S3 IP; the *reply* tuple shows the pod's address rewritten to the node IP (`192.0.2.10`) — that's the masquerade, and the fact that the reverse mapping lives *in this row* is why NAT needs conntrack.

For a large, busy table, `conntrack -L` is expensive: it's a full table dump every time you poll it. `conntrack -E` (event mode) instead streams create/update/destroy events as they happen — a lighter-weight way to watch flow churn on a hot node without repeatedly paying the dump cost.

### 2. Why pod egress NAT is one conntrack entry per flow

Pod IPs (`10.244.x.x`) aren't routable on the physical network. To reach the internet, the CNI (or kube-proxy) installs a `MASQUERADE`/`SNAT` rule in `POSTROUTING` that rewrites the source to the node's IP. For the return packet to find its way back to the right pod and port, the kernel must remember the translation — that memory *is* the conntrack entry. So:

- **One flow = one conntrack tuple.** A dataloader that opens 500 concurrent HTTPS connections to fetch shards = 500 entries. Multiply by pods per node.
- **Short-lived flows linger.** A closed TCP connection doesn't free its entry immediately; it sits in `TIME_WAIT` for `nf_conntrack_tcp_timeout_time_wait` (default **120 s**). A burst of 10k curls that each last 50 ms can pin ~10k entries for two minutes *after* they finish.
- **The table is node-global.** Every pod's egress shares one `nf_conntrack_max`. A single noisy pod (a crawler, a broken retry loop, a metrics agent hammering a collector) can exhaust it for the whole node, taking out unrelated workloads. This is a classic GPU-node failure: NCCL is fine (RDMA), but the training script's S3 prefetch fan-out blows conntrack and the *checkpoint upload* mysteriously times out.

Scaling math to internalize: at 262144 max and a 120 s TIME_WAIT, sustained new-connection rate that fills it is ~2180 conn/s of purely short-lived flows before you're at the ceiling — a single busy dataloader node reaches that.

### 3. Exhaustion: the symptom, the log line, the sysctls

When `nf_conntrack_count` hits `nf_conntrack_max`, new-flow packets that would need a new entry are **dropped** and the kernel rate-limits this to dmesg:

```
$ dmesg | grep -i conntrack
[ 8842.115] nf_conntrack: nf_conntrack: table full, dropping packet
```

Crucially, **established flows keep working** — only packets that require a *new* entry (i.e., new connections, the `NEW` state) are dropped. So the operator-visible symptom sequence is:

1. First: **rising latency and timeouts on new connections** while existing ones are fine. DNS lookups (each query can be a new conntrack entry for UDP) start failing intermittently — often the first canary.
2. Then: `table full, dropping packet` in dmesg, and application logs full of connection timeouts / TLS handshake failures — never clean RSTs, because the SYN is silently dropped, so clients wait the full connect timeout.
3. Retries make it *worse*: each retry is another new-flow attempt hammering a full table.

Diagnose and tune (persist via a `sysctl.d` drop-in or the CNI's config, not just runtime):

```
# how close to the edge:
$ awk '{print}' /proc/sys/net/netfilter/nf_conntrack_count
$ sysctl net.netfilter.nf_conntrack_max

# raise the ceiling (memory: ~size of a struct nf_conntrack, ~300 B/entry):
$ sysctl -w net.netfilter.nf_conntrack_max=1048576
# hash table buckets should scale too (default max/8 historically):
$ echo 262144 > /sys/module/nf_conntrack/parameters/hashsize

# reclaim faster — shrink the timeouts that pin dead flows:
$ sysctl -w net.netfilter.nf_conntrack_tcp_timeout_time_wait=30
$ sysctl -w net.netfilter.nf_conntrack_tcp_timeout_close_wait=30
$ sysctl -w net.netfilter.nf_conntrack_udp_timeout=15
```

Trade-off: raising `nf_conntrack_max` costs RAM (~300 B/entry → 1M entries ≈ 300 MB non-swappable kernel memory, plus the hash table); shrinking timeouts frees entries faster but risks reusing a tuple while a straggler packet is in flight. On a fleet, monitor `nf_conntrack_count / nf_conntrack_max` as a ratio and alert at ~80%. `conntrack -S` shows per-CPU stats including `insert_failed` and `drop` counters — non-zero `drop` is the smoking gun even before dmesg fills.

```
$ conntrack -S
cpu=0  found=... insert=... insert_failed=12  drop=12  early_drop=0  ...
```

### 4. Why eBPF datapaths (Cilium) reduce or remove conntrack load

kube-proxy in iptables mode implements Services as chains of NAT rules; every serviced or masqueraded flow lands a netfilter conntrack entry, and rule evaluation is O(rules). At thousands of Services × endpoints, both the rule set and the conntrack pressure balloon.

Cilium's eBPF datapath changes the mechanism:

- **Socket-level load balancing (socket-LB).** For traffic *originating on the node* (pod → ClusterIP), a BPF program attached to the socket (`connect(2)`/`sendmsg` hooks, cgroup/bpf) rewrites the destination to a backend pod IP **at connect time, before a packet exists**. There's no per-packet DNAT in `POSTROUTING`, so no netfilter conntrack entry is created for the service translation — the connection is made directly to the backend. This removes an entire class of conntrack load and the iptables rule walk.
- **BPF host routing / DSR.** For north-south and pod-to-pod, Cilium can bypass the upper netfilter path (`bpf_redirect` at tc/XDP) and, with Direct Server Return, avoid return-path SNAT state on the LB node.
- **Its own eBPF conntrack map (when needed).** Where connection state *is* required (e.g., stateful policy, SNAT for masquerade), Cilium keeps state in a BPF hash map it owns, sized and aged independently of `nf_conntrack_max`, and can disable the kernel's `nf_conntrack` for that traffic entirely. You can run with `bpf.masquerade=true` so masquerade happens in eBPF, not iptables.

Net effect for a GPU node: fewer or zero netfilter conntrack entries for Service and (optionally) egress traffic, no O(n) iptables rule walk, and per-node observability via `cilium bpf ct list` / Hubble instead of `conntrack -L`. The failure mode from §3 shrinks because the table that used to fill (`nf_conntrack`) is no longer on the hot path — though you must now monitor Cilium's BPF CT map size instead. It's not magic; it's moving the state into a purpose-built, node-owned map and doing translation before packets exist. (Lesson 08 explains exactly what a "BPF map" and "socket hook" are, mechanically — this lesson only needs you to know that they exist and what they replace.)

## Perspectives

**Kernel-mechanism view.** The whole chapter reduces to one coupling: NAT cannot exist without conntrack, because the reverse translation has to live *somewhere*, and that somewhere is the conntrack entry created at `PREROUTING`/`OUTPUT` and confirmed at `POSTROUTING`. Every other fact in this lesson — per-flow entries, TIME_WAIT lingering, table exhaustion — falls out of that one structural fact.

**Operator/SRE view.** The symptom sequence in §3 is the actual on-call pattern-match, not a footnote. "Existing connections are fine, new connections hang or time out, and it's happening on our busiest nodes" is a fingerprint you should recognize in under a minute — before you've even opened `conntrack -S`. Most engineers reach for "check the network," which wastes the first 20 minutes of an incident on the wrong layer. The experienced move is to check `nf_conntrack_count / nf_conntrack_max` and `conntrack -S drop` *first*, precisely because the symptom is so easy to mistake for a network problem.

**GPU-fleet-specific view.** This is the headline case for this module: distributed training uses NCCL over RDMA for the actual gradient traffic, which never touches the host TCP/IP stack — so on-call engineers reflexively rule out "the network" when NCCL health checks look fine. But checkpoint uploads, dataset prefetch from object storage, and control-plane calls are ordinary TCP through the host stack, fully subject to conntrack. The result is a genuinely confusing incident shape: **"NCCL is fine, but checkpoint uploads intermittently time out"** — two traffic classes sharing a node, only one of them protected from the shared kernel resource, and the failure looks like it should be impossible given "the network is healthy." Recognizing that RDMA bypass and conntrack exhaustion are *orthogonal* is the specific expertise this role pays for.

**Economics/architecture view.** Step back from the mechanism and the conntrack-vs-eBPF choice is really a **state-ownership and blast-radius decision**, not just a scaling knob. A node-global `nf_conntrack` table means every pod on that node — regardless of tenant, team, or workload class — shares one finite resource with no per-pod isolation. One noisy pod (a broken retry loop, an unbounded crawler, a misconfigured sidecar) can starve every other tenant's NAT'd egress on that node, even though nothing about *their* traffic changed. That's a fairness failure, not merely a "we hit a limit" failure — the conntrack table has no notion of quota or priority per pod. Cilium's eBPF conntrack map doesn't just move the bottleneck somewhere with more headroom; it changes what's possible to isolate and account for per workload. This is the framing a staff-level design review actually probes: not "does raising `nf_conntrack_max` fix the incident" but "does our datapath let one tenant blast-radius the rest of the node, and what's the architectural fix versus the tactical one."

## Real-world use cases

- **[The Cloudflare Blog — "Conntrack turns a blind eye to dropped SYNs"](https://blog.cloudflare.com/conntrack-turns-a-blind-eye-to-dropped-syns/)** — a deep Cloudflare investigation into how conntrack handles SYN packets under contention, and how drops can happen in ways invisible to conntrack's own accounting. Shows that "check `conntrack -S drop`" isn't always the whole story — the failure can hide even deeper in the SYN-handling path.
- **[Mark Betz (Medium) — "Exhausting conntrack table space crippled our k8s cluster"](https://medium.com/@betz.mark/exhausting-conntrack-table-space-crippled-our-k8s-cluster-98564f6f34e0)** — a widely-cited independent postmortem (not a company blog) describing service-mesh/proxy connection churn filling a node-global conntrack table in production Kubernetes. This is the canonical "conntrack table full" story and matches the failure mode described in §2–3 almost exactly.

## Worked example

**Investigation: "checkpoint uploads from training pods intermittently time out; NCCL is fine."**

1. **Symptom triage.** NCCL over RDMA is healthy (bypasses the host stack), which rules out fabric. Failing thing is TCP egress to object storage — and it's *new* connections timing out, not resets. That shape screams conntrack.

2. **Measure headroom.** On the affected node:
   ```
   $ sysctl net.netfilter.nf_conntrack_count net.netfilter.nf_conntrack_max
   net.netfilter.nf_conntrack_count = 258900
   net.netfilter.nf_conntrack_max   = 262144      # 98.8% full
   $ conntrack -S | grep -o 'drop=[0-9]*' | head
   drop=4471                                        # real drops, not zero
   $ dmesg | grep -c 'table full'
   37
   ```
   Confirmed: table is saturated and dropping.

3. **Find the culprit.** `conntrack -L -p tcp --dport 443 | wc -l` and group by source pod IP:
   ```
   $ conntrack -L 2>/dev/null | awk '{print $5}' | sort | uniq -c | sort -rn | head
   198431 src=10.244.7.9      # one dataloader pod owns 75% of the table
   ```
   That pod's prefetcher opens hundreds of concurrent short-lived S3 connections per batch, and TIME_WAIT (120 s) keeps them resident long after they close.

4. **Immediate mitigation.** Buy headroom and reclaim faster:
   ```
   $ sysctl -w net.netfilter.nf_conntrack_max=1048576
   $ sysctl -w net.netfilter.nf_conntrack_tcp_timeout_time_wait=30
   ```
   `nf_conntrack_count` ratio drops well below the ceiling; checkpoint timeouts stop. Persist in `/etc/sysctl.d/`.

5. **Real fix.** (a) App-side: make the dataloader reuse a connection pool / HTTP keep-alive instead of a new connection per object — collapses hundreds of flows into a handful. (b) Platform-side: move the node to Cilium with `kube-proxy` replacement and eBPF masquerade so Service traffic and egress SNAT leave the `nf_conntrack` table, and monitor `cilium bpf ct list global | wc -l` instead. Root cause read: the incident was never "flaky network" — it was a node-global, per-flow-NAT resource exhausted by one pod's connection churn, exactly the fairness failure described in the economics perspective above.

## Practice

**Environment:** a Linux VM or container where you can safely load traffic (not a shared prod node). Root/`CAP_NET_ADMIN` needed for `conntrack`/`sysctl`. Install `conntrack-tools` and `curl`/`nc`.

**Feeds:** [Anatomy of a Container](../practice/anatomy-of-a-container/README.md) — commit your findings and the sysctl playbook into its diagnostic toolkit.

1. **Baseline.** Record `sysctl net.netfilter.nf_conntrack_count net.netfilter.nf_conntrack_max` and `conntrack -S` (note `drop`/`insert_failed`). Dump a few live entries: `conntrack -L | head`.
2. **Generate short-lived flows.** In a loop, hammer new connections to a target you control or a local server:
   ```
   # start a throwaway listener/server, or curl a local nginx
   for i in $(seq 1 5000); do
     curl -s -o /dev/null http://127.0.0.1/ &
   done; wait
   ```
   (Localhost still creates conntrack entries when a nat/masquerade rule or `nf_conntrack` is loaded; if not, target a pod-networked service or use `nc` across a NATed bridge.)
3. **Watch the count climb.** In another shell: `watch -n1 'cat /proc/sys/net/netfilter/nf_conntrack_count'`. Observe it rise, then decay over ~120 s as TIME_WAIT entries expire. Note the decay rate. Optionally, run `conntrack -E` in a third shell and compare how it feels to watch events stream versus polling `conntrack -L`.
4. **Read the exhaustion symptom.** *Safely*, temporarily set a tiny ceiling on a disposable VM: `sysctl -w net.netfilter.nf_conntrack_max=256`, re-run the loop, and catch `dmesg | grep conntrack` → `table full, dropping packet`, plus rising `drop=` in `conntrack -S`. Restore the ceiling afterward. (If you can't safely fill it, reason in your note about the exact log line and which connections fail vs. survive.)
5. **Identify the tuning.** Decide the two sysctls you'd change and justify the RAM cost of raising `nf_conntrack_max`.

**Acceptance:** a note that shows (1) `nf_conntrack_count`/`max` at baseline and under load (the count climbing), (2) the exact exhaustion symptom — the `nf_conntrack: table full, dropping packet` line and/or non-zero `conntrack -S` `drop`, and the statement "existing connections survive, new ones are dropped," and (3) the specific sysctl(s) you'd tune with the trade-off (RAM per entry vs. tuple-reuse risk).

## Common pitfalls

1. **Assuming a "flaky network" symptom is a network-layer problem.** Intermittent new-connection timeouts, especially clustered on your busiest nodes, are frequently a node-local kernel resource (the conntrack table) — not packet loss on the wire. Check `nf_conntrack_count/max` and `conntrack -S drop` before escalating to the network team.
2. **Believing raising `nf_conntrack_max` is free.** It costs non-swappable kernel memory (~300 B/entry) and the hash table scales with it too (`hashsize`). At 1M entries that's roughly 300 MB of RAM you can never reclaim under memory pressure — budget it explicitly on GPU nodes where memory is already contended by hugepages and pinned buffers.
3. **Assuming RDMA/NCCL traffic is protected from conntrack exhaustion.** RDMA bypasses the host network stack entirely, so it's genuinely immune — but control-plane TCP traffic (health checks, checkpoint uploads, metrics) on the *same node* is not, and shares the same table as everything else on the box.
4. **Thinking "Cilium replaces kube-proxy" means "no more conntrack."** Cilium keeps its own eBPF conntrack map wherever stateful policy or masquerade needs state — it shrinks the problem by moving it off `nf_conntrack` and giving you a purpose-built map, but it doesn't eliminate connection tracking as a concept. You still need to monitor that map's size.
5. **Shortening TIME_WAIT timeouts without understanding the reuse-risk trade-off.** Freeing entries faster buys headroom, but it also risks reusing a tuple (source IP/port pair) while a delayed or retransmitted packet from the old connection is still in flight — which can corrupt or misdeliver data to the new connection. Treat it as a real trade-off, not a free win.

## Self-check

- **What produces "nf_conntrack: table full, dropping packet," and what user-visible symptoms precede it?**
  **Answer:** It's logged when `nf_conntrack_count` reaches `nf_conntrack_max` and a packet arrives that would require a *new* conntrack entry (a NEW-state flow) with no room to insert it — so the kernel drops that packet. Because only new-entry packets are dropped while established flows keep their existing entries, the symptoms that precede the log line are: intermittent timeouts and rising latency on *new* connections while existing ones work fine, DNS lookups failing sporadically (UDP queries can each need an entry), and TLS/connect timeouts rather than clean RSTs (the SYN is silently dropped, so clients wait the full connect timeout). Retries worsen it. `conntrack -S` shows non-zero `drop`/`insert_failed` even before dmesg, making it a good early signal along with the `count/max` ratio crossing ~80%.

- **Why does pod egress NAT create a conntrack entry per flow, and how does that scale on a busy node?**
  **Answer:** Pod IPs aren't routable on the physical network, so the CNI/kube-proxy installs a MASQUERADE/SNAT rule in POSTROUTING that rewrites the source to the node IP. For return packets to be reverse-translated back to the correct pod and port, the kernel must store the mapping — and that storage *is* the conntrack entry; NAT is implemented on top of conntrack, so every NATed flow requires exactly one tuple. It scales badly because the table is node-global (every pod shares one `nf_conntrack_max`), each short-lived connection lingers in TIME_WAIT (~120 s default) long after closing, and connection-churny workloads (dataloaders opening hundreds of concurrent S3 connections per batch) generate entries far faster than they drain. A single noisy pod can exhaust the whole node's table and take out unrelated workloads — a common GPU-node failure where RDMA/NCCL is fine but TCP egress (checkpoint upload) times out.

- **How does Cilium's eBPF datapath reduce or remove conntrack load vs. kube-proxy/iptables?**
  **Answer:** kube-proxy's iptables mode implements Services as NAT rule chains, so every serviced/masqueraded flow lands a netfilter conntrack entry and rules are walked O(n). Cilium moves the mechanism into eBPF: with **socket-LB**, a BPF program on the socket's connect/sendmsg hooks rewrites the destination to a backend pod IP *at connect time, before any packet exists*, so there's no per-packet DNAT in POSTROUTING and no netfilter conntrack entry for the Service translation. With **BPF host routing/DSR** it bypasses the upper netfilter path and avoids return-path SNAT state. Where connection state is genuinely needed (stateful policy, egress masquerade with `bpf.masquerade=true`), Cilium keeps it in its own BPF hash map, sized and aged independently of `nf_conntrack_max`, and can disable the kernel's `nf_conntrack` for that traffic. Net: fewer/zero netfilter conntrack entries and no O(n) rule walk — but you now monitor Cilium's BPF CT map (`cilium bpf ct list`, Hubble) instead of `conntrack -L`.

- **Why is conntrack exhaustion better framed as a "fairness failure" than a "scaling limit"?**
  **Answer:** A scaling-limit framing implies the fix is always "raise the ceiling." A fairness framing recognizes that `nf_conntrack_max` is a single, node-global, unpartitioned resource with no per-pod quota — so one noisy or misbehaving pod can consume nearly all of it and starve every other tenant on the node, even though those other tenants did nothing wrong. Raising the ceiling buys headroom but doesn't fix the underlying lack of isolation; the architectural fix (per-workload quotas, or moving state into a datapath that can account per-identity, like Cilium's eBPF conntrack map) addresses the actual blast-radius problem. This is the framing a staff-level design review probes: not just "did we hit a limit" but "should one workload be able to do this to its neighbors at all."

- **What's the advantage of `conntrack -E` over repeatedly polling `conntrack -L` on a large table?**
  **Answer:** `conntrack -L` dumps the entire table on every invocation, which is expensive when the table has hundreds of thousands of entries and you're polling it frequently to watch churn. `conntrack -E` instead switches to event mode and streams create/update/destroy events as they happen in real time, so you see flow lifecycle changes without repeatedly paying the cost of a full table dump — a lighter-weight way to observe a busy node.

## Connections & what's next

This lesson sits alongside lesson 03 (cgroups v2) and lesson 04 (PSI) in the module's larger theme: fleet-scale failures come from **shared, finite kernel resources** that look fine in aggregate and fail invisibly per-tenant — cgroup limits, PSI stall, hugepage/NUMA locality, and now conntrack table space are all instances of the same pattern, just in different subsystems. It also sharpens lesson 06's memory-side framing into a network-side one, and it sets up the eBPF story that Cilium leans on for its datapath.

Next: **08 — eBPF**, which explains the general mechanism (verifier, JIT, hooks, maps) behind the socket-LB and BPF conntrack map this lesson took on faith. The thread that carries forward is direct: once you understand what a BPF program attached to a socket hook actually *is*, "Cilium removes conntrack load" stops being a vendor claim and becomes a mechanism you can verify yourself with `bpftool` and `cilium bpf ct list`.

## References & further reading

**Primary sources**
- [`nf_conntrack-sysctl` kernel docs](https://www.kernel.org/doc/Documentation/networking/nf_conntrack-sysctl.txt) — the authoritative list of every timeout and limit knob (`nf_conntrack_max`, `_tcp_timeout_time_wait`, `hashsize`). Read for the exact sysctl semantics you tune under incident pressure.
- [Kubernetes docs — Service virtual IPs and kube-proxy](https://kubernetes.io/docs/reference/networking/virtual-ips/) — how kube-proxy's Service/masquerade modes translate into the conntrack pressure described in §2. Read for how Service NAT maps onto real pod-egress patterns.

**Real-world engineering blogs**
- [The Cloudflare Blog — "Conntrack turns a blind eye to dropped SYNs"](https://blog.cloudflare.com/conntrack-turns-a-blind-eye-to-dropped-syns/) — what it shows: conntrack's SYN handling under contention can drop packets in ways invisible to conntrack's own counters.
- [Mark Betz (Medium) — "Exhausting conntrack table space crippled our k8s cluster"](https://medium.com/@betz.mark/exhausting-conntrack-table-space-crippled-our-k8s-cluster-98564f6f34e0) — what it shows: the canonical conntrack-table-full postmortem, service-mesh connection churn filling a node-global table (widely-cited independent account, not an official company blog).

**Deeper dives**
- [Cilium documentation — kube-proxy replacement & eBPF datapath](https://docs.cilium.io/) (see "kube-proxy replacement," "socket LB," and the eBPF datapath concept pages) — the strategic fix for conntrack scaling on GPU nodes, and a strong cost/observability argument in a platform design review.
