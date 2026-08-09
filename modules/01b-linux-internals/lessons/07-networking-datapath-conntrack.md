---
lesson: "01b.7"
title: "The network datapath: netfilter, conntrack, and NAT at scale"
module: "01b"
concept: "The network datapath: netfilter, conntrack, and NAT at scale"
status: not-started
est_time: "5h"
artifacts: []
---

# 01b.7 · The network datapath: netfilter, conntrack, and NAT at scale

> **Concept.** Every NATed pod flow leaves a fingerprint in the kernel's connection-tracking table; when that table fills, a busy node drops packets silently — and eBPF datapaths exist largely to escape it.
>
> Module: [🐧 01b — Linux systems internals](../README.md) · Deliverable: [Anatomy of a Container](../practice/anatomy-of-a-container/README.md)

## Why this matters

On a GPU cluster the network *is* the workload. Distributed training runs NCCL all-reduce across nodes; checkpointing and data ingestion pull terabytes from object storage; sidecars phone home to control planes. Two traffic shapes stress the kernel differently, and both route through the same chokepoint — **conntrack**:

- **NCCL / RDMA** ideally bypasses the host TCP/IP stack (RoCE/IB), but the control channels, health checks, and any TCP fallback still traverse netfilter.
- **Egress fan-out** — thousands of short-lived HTTPS connections from dataloaders scraping shards out of S3/GCS, metrics push, log shipping — creates a *new conntrack entry per flow*, and because pods egress via source-NAT (masquerade) to the node IP, the node's conntrack table is the shared resource that all pods contend for.

When `nf_conntrack_max` fills, the kernel logs `nf_conntrack: table full, dropping packet` and starts dropping *new* connections. The user-visible symptom is maddening: existing connections work, but new ones hang and time out — intermittently, under load, on your busiest nodes, exactly when a training job ramps its data pipeline. Knowing this mechanism is the difference between "the network is flaky" and "node conntrack is at 98%, here's the sysctl and here's why we should move to an eBPF datapath." That's a cost/observability differentiator: you catch it in a dashboard before it pages.

## From using to understanding

**What you already know as an operator:**
- Pods reach the internet; kube-proxy programs Services; you've maybe bumped `net.netfilter.nf_conntrack_max` after an incident.
- You've seen `iptables -t nat -L` produce a wall of rules and `conntrack -L` dump connections.
- You know Cilium "replaces kube-proxy" and is "eBPF-based" but not mechanically why that helps.

**What the kernel is actually doing:**
- Packets traverse **netfilter hooks** — five fixed points in the network stack (`PREROUTING`, `INPUT`, `FORWARD`, `OUTPUT`, `POSTROUTING`) where iptables/nftables rules and the conntrack module run.
- **conntrack** (`nf_conntrack`) is a stateful table: the first packet of a flow creates a tuple entry; subsequent packets are matched against it so the kernel knows a flow's state (NEW/ESTABLISHED/RELATED) and can reverse-NAT reply packets.
- **NAT is implemented on top of conntrack** — the translation is *stored in the conntrack entry*, so NAT is impossible without a conntrack entry per flow. This is the coupling that makes conntrack the scaling bottleneck.
- **eBPF datapaths** run a program at the socket or driver (XDP/tc) layer that does load-balancing and (with DSR or per-socket LB) avoids inserting per-packet netfilter NAT state — sidestepping the table.

## Core notes

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

Net effect for a GPU node: fewer or zero netfilter conntrack entries for Service and (optionally) egress traffic, no O(n) iptables rule walk, and per-node observability via `cilium bpf ct list` / Hubble instead of `conntrack -L`. The failure mode from §3 shrinks because the table that used to fill (`nf_conntrack`) is no longer on the hot path — though you must now monitor Cilium's BPF CT map size instead. It's not magic; it's moving the state into a purpose-built, node-owned map and doing translation before packets exist.

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

5. **Real fix.** (a) App-side: make the dataloader reuse a connection pool / HTTP keep-alive instead of a new connection per object — collapses hundreds of flows into a handful. (b) Platform-side: move the node to Cilium with `kube-proxy` replacement and eBPF masquerade so Service traffic and egress SNAT leave the `nf_conntrack` table, and monitor `cilium bpf ct list global | wc -l` instead. Root cause read: the incident was never "flaky network" — it was a node-global, per-flow-NAT resource exhausted by one pod's connection churn.

## Practice

**Environment:** a Linux VM or container where you can safely load traffic (not a shared prod node). Root/`CAP_NET_ADMIN` needed for `conntrack`/`sysctl`. Install `conntrack-tools` and `curl`/`nc`.

1. **Baseline.** Record `sysctl net.netfilter.nf_conntrack_count net.netfilter.nf_conntrack_max` and `conntrack -S` (note `drop`/`insert_failed`). Dump a few live entries: `conntrack -L | head`.
2. **Generate short-lived flows.** In a loop, hammer new connections to a target you control or a local server:
   ```
   # start a throwaway listener/server, or curl a local nginx
   for i in $(seq 1 5000); do
     curl -s -o /dev/null http://127.0.0.1/ &
   done; wait
   ```
   (Localhost still creates conntrack entries when a nat/masquerade rule or `nf_conntrack` is loaded; if not, target a pod-networked service or use `nc` across a NATed bridge.)
3. **Watch the count climb.** In another shell: `watch -n1 'cat /proc/sys/net/netfilter/nf_conntrack_count'`. Observe it rise, then decay over ~120 s as TIME_WAIT entries expire. Note the decay rate.
4. **Read the exhaustion symptom.** *Safely*, temporarily set a tiny ceiling on a disposable VM: `sysctl -w net.netfilter.nf_conntrack_max=256`, re-run the loop, and catch `dmesg | grep conntrack` → `table full, dropping packet`, plus rising `drop=` in `conntrack -S`. Restore the ceiling afterward. (If you can't safely fill it, reason in your note about the exact log line and which connections fail vs. survive.)
5. **Identify the tuning.** Decide the two sysctls you'd change and justify the RAM cost of raising `nf_conntrack_max`.

**Acceptance:** a note that shows (1) `nf_conntrack_count`/`max` at baseline and under load (the count climbing), (2) the exact exhaustion symptom — the `nf_conntrack: table full, dropping packet` line and/or non-zero `conntrack -S` `drop`, and the statement "existing connections survive, new ones are dropped," and (3) the specific sysctl(s) you'd tune with the trade-off (RAM per entry vs. tuple-reuse risk).

## Self-check

**(a) What produces "nf_conntrack: table full, dropping packet," and what user-visible symptoms precede it?**

**Answer:** It's logged when `nf_conntrack_count` reaches `nf_conntrack_max` and a packet arrives that would require a *new* conntrack entry (a NEW-state flow) with no room to insert it — so the kernel drops that packet. Because only new-entry packets are dropped while established flows keep their existing entries, the symptoms that precede the log line are: intermittent timeouts and rising latency on *new* connections while existing ones work fine, DNS lookups failing sporadically (UDP queries can each need an entry), and TLS/connect timeouts rather than clean RSTs (the SYN is silently dropped, so clients wait the full connect timeout). Retries worsen it. `conntrack -S` shows non-zero `drop`/`insert_failed` even before dmesg, making it a good early signal along with the `count/max` ratio crossing ~80%.

**(b) Why does pod egress NAT create a conntrack entry per flow, and how does that scale on a busy node?**

**Answer:** Pod IPs aren't routable on the physical network, so the CNI/kube-proxy installs a MASQUERADE/SNAT rule in POSTROUTING that rewrites the source to the node IP. For return packets to be reverse-translated back to the correct pod and port, the kernel must store the mapping — and that storage *is* the conntrack entry; NAT is implemented on top of conntrack, so every NATed flow requires exactly one tuple. It scales badly because the table is node-global (every pod shares one `nf_conntrack_max`), each short-lived connection lingers in TIME_WAIT (~120 s default) long after closing, and connection-churny workloads (dataloaders opening hundreds of concurrent S3 connections per batch) generate entries far faster than they drain. A single noisy pod can exhaust the whole node's table and take out unrelated workloads — a common GPU-node failure where RDMA/NCCL is fine but TCP egress (checkpoint upload) times out.

**(c) How does Cilium's eBPF datapath reduce or remove conntrack load vs. kube-proxy/iptables?**

**Answer:** kube-proxy's iptables mode implements Services as NAT rule chains, so every serviced/masqueraded flow lands a netfilter conntrack entry and rules are walked O(n). Cilium moves the mechanism into eBPF: with **socket-LB**, a BPF program on the socket's connect/sendmsg hooks rewrites the destination to a backend pod IP *at connect time, before any packet exists*, so there's no per-packet DNAT in POSTROUTING and no netfilter conntrack entry for the Service translation. With **BPF host routing/DSR** it bypasses the upper netfilter path and avoids return-path SNAT state. Where connection state is genuinely needed (stateful policy, egress masquerade with `bpf.masquerade=true`), Cilium keeps it in its own BPF hash map, sized and aged independently of `nf_conntrack_max`, and can disable the kernel's `nf_conntrack` for that traffic. Net: fewer/zero netfilter conntrack entries and no O(n) rule walk — but you now monitor Cilium's BPF CT map (`cilium bpf ct list`, Hubble) instead of `conntrack -L`.

## Resources

1. **conntrack-tools & `nf_conntrack` sysctl reference** — `man 8 conntrack` and the kernel `nf_conntrack-sysctl` docs (https://www.kernel.org/doc/Documentation/networking/nf_conntrack-sysctl.txt) — the authoritative list of every timeout and limit knob (`nf_conntrack_max`, `_tcp_timeout_time_wait`, `hashsize`) plus the `conntrack -L/-S` output format. **Deep** on the sysctl list once, **skim** the man page. Why: you tune these under incident pressure; know the RAM cost and the TIME_WAIT trade-off cold.
2. **"Kubernetes & conntrack" / iptables-at-scale writeup** — e.g. the classic conntrack-exhaustion postmortems and the Kubernetes network docs on kube-proxy modes (https://kubernetes.io/docs/reference/networking/virtual-ips/) — how Service/masquerade rules translate into conntrack pressure on real nodes. **Skim.** Why: connects the kernel mechanism to the exact pod-egress and Service patterns that fill the table on your fleet.
3. **Cilium documentation — kube-proxy replacement & eBPF datapath** — https://docs.cilium.io/ (see "kube-proxy replacement," "socket LB," and the eBPF datapath concept pages) — how socket-LB and BPF masquerade remove per-packet netfilter NAT/conntrack. **Skim** the concept pages, **deep** if you're evaluating a migration. Why: this is the strategic fix for conntrack scaling on GPU nodes and a strong cost/observability argument in a platform design review.
