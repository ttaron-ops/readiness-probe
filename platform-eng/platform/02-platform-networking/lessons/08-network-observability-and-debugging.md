---
lesson: "A02.8"
title: "Network observability and debugging"
module: "A-02"
concept: "disciplined bisection + eBPF flow telemetry"
status: not-started
est_time: "4 hrs"
prev: "07-gpu-and-rdma-networking.md"
next: null
artifacts: ["network-debugging decision-tree runbook"]
sources: 13
---

# A02.8 · Network observability and debugging

> **Concept.** Staff network debugging is disciplined bisection down a decision tree, not tool-flailing — and at fleet scale the tree runs on identity-based eBPF flow telemetry, not IP-based packet sampling.
>
> Module: [🌐 Platform networking depth](../README.md) · Track A — Platform excellence

## Where this fits

This lesson closes the module's through-line. Every prior lesson put a mechanism, a cost, and a failure mode on one hop: the kernel datapath (01), DNS (02), the load balancer (03), the cloud boundary (04), the Kubernetes dataplane (05), the mesh (06), and the deliberately-bypassed RDMA path (07). None of that helps in an incident unless you can bisect *across* all of them fast, under pressure, without knowing in advance which hop is guilty.

That is the actual staff skill this module builds toward, and it is also the checkpoint's format: given a symptom, name the exact mechanism and the exact command. So this lesson does not add a hop. It adds **a procedure that decides which hop to look at next, and a toolkit that can answer each question with one command** — plus an honest map of where that toolkit's vision ends, which is exactly at the boundary lesson 07 defined.

## Why this matters

The senior-to-staff jump here is not more tools. It is a procedure that **converges** — that reaches a verdict in a bounded number of steps, and where each step reduces the remaining search space regardless of its outcome.

The failure mode being corrected is a specific one, and everybody has done it: you get "intermittent timeouts in prod", you open a terminal, and you tcpdump. Forty minutes later you have four gigabytes of pcap, three theories, and no verdict, because packet capture answers "what did the wire carry" and the question was "which of eight subsystems is broken". The staff move is to spend the first sixty seconds *classifying the symptom* and the next five minutes running four one-command tests that each eliminate a whole branch.

There is a second, structural reason. At fleet scale the old dataplane — sample the flows, grep the IPs — is broken, because a pod IP is valid for the life of a pod, which is minutes. By the time a sampled, delayed flow record surfaces `10.0.3.7 → 10.0.9.2`, both pods may be gone and both addresses reassigned to different workloads in different namespaces. **The unit of debugging has to be identity, not address**, and the tooling that provides identity is the eBPF dataplane's, not the cloud's.

## What's new here (calibration)

- **Assumed known and not re-taught:** how to run tcpdump and write BPF filters, ping/traceroute/mtr, what a cloud flow log is, that eBPF can attach to the kernel, and basic "is the port even open" triage. The counter-per-datapath-stage table (`ethtool -S`, `/proc/net/softnet_stat`, `tc -s qdisc`, `nstat`) is A02.1 §5 and is used here, not repeated. Conntrack's internals, its counters and the DNS insert race are 01b.7 and are used the same way.
- **New: the procedure treated as an algorithm** — what makes a branch test *good*, how to order the branches, and why "rule out" beats "confirm". This is transferable well beyond networking, and it is what interviewers are actually testing when they ask "how would you debug X".
- **New: symptom fingerprinting.** Six symptom *shapes* — quantisation, size-dependence, load-dependence, one-sidedness, correlation with a deploy, and error-free slowness — each of which picks a branch before you run anything.
- **New: reading `ss -tie` as a diagnostic instrument**, field by field, including the three fields (`busy`, `rwnd_limited`, `sndbuf_limited`) that exist in iproute2's source but not in its man page, and which are the difference between "the network is slow" and "your application is slow".
- **New: the kernel's own drop taxonomy.** Since the `skb:kfree_skb` tracepoint gained a `reason` argument, the kernel names *why* it discarded a packet, from an enum of 200+ values. That turns "packets are disappearing" from an inference into a lookup.
- **New: what Hubble's flow record actually contains** — the verdict enum, the drop-reason enum, and the policy-correlation fields that name the specific object that denied a flow — and what it cannot tell you.
- **New: `pwru`'s real flag surface**, including the flags that solve the encapsulation problem the older literature describes as a hard limitation.
- **New: the toolkit's boundary, stated precisely** — the mesh (which sees L7 but not packets), the cloud LB (which is opaque), and the RDMA fabric (which the kernel never sees).

## Core concepts

### 1. What makes a procedure converge

Debugging is search. A procedure converges when every step shrinks the space of possible causes by a large factor regardless of which way the step comes out. That gives three properties a good branch test must have, and they are worth being explicit about because most people optimise for the wrong one:

1. **High specificity, not high sensitivity.** You want a test whose *negative* result is trustworthy. "DNS resolves fine when I run `dig` once" is a weak negative — the failure is intermittent. "Latency is not quantised at any multiple of 5 s across 10,000 samples" is a strong negative, because the DNS retry mechanism *cannot* produce a non-quantised delay.
2. **Cheap and side-effect-free.** A test that needs a new DaemonSet, a privileged pod, or a config change is not a branch test; it is a project. Order by cost, and prefer tests that read state you already have.
3. **Rules out a branch, not a leaf.** "Is it DNS?" is a branch. "Is it this specific CoreDNS pod?" is a leaf. Testing leaves early is how a bisection turns into a random walk.

From those, the ordering rule: **run tests in increasing order of cost divided by the fraction of the search space they eliminate.** In practice, on a Kubernetes cluster, that ordering is remarkably stable across incidents, and it is worth memorising as a default:

```
   ORDER OF BRANCHES, and why this order

   0. Classify the symptom shape          cost: 0      eliminates: ~half
      (no commands — just look at the latency distribution and the
       failure ratio; §2 turns this into a table)

   1. Is it the client's own socket?      cost: 1 cmd  eliminates: everything
      ss -tie inside the affected pod.     below the socket, in one shot.
      If retrans=0, rtt is normal, and the socket is receiver- or
      app-limited, the network delivered every byte. STOP.

   2. Is it name resolution?              cost: 1 cmd  eliminates: DNS branch
      Quantisation at 5 s multiples is the fingerprint; the test is a
      timed lookup loop plus resolv.conf. (Mechanism: 02, 01b.7.)

   3. Is it path MTU?                     cost: 1 cmd  eliminates: MTU branch
      Size-dependence is the fingerprint; the test is a DF-bit sweep.
      (Mechanism and the three fixes: 05 §5.)

   4. Is it the dataplane dropping it?    cost: 1 cmd  eliminates: L3/L4 branch
      hubble observe --verdict DROPPED, or the kfree_skb reason (§6).
      This is the first test that needs cluster tooling to exist.

   5. Is it the LB or the mesh?           cost: 1 cmd  eliminates: L7 branch
      Envoy response flags / upstream stats. (Vocabulary: 06 §7.)

   6. Is it the fabric?                   cost: 3 cmds eliminates: nothing else
      Only reachable after 1–5 are clean, and this lesson's tools cannot
      see it at all (§10). Hand off to 07's acceptance signals.
```

**The single most valuable habit in that list is step 1**, and it is the one people skip. A large fraction of "the network is broken" tickets are resolved by a socket that shows zero retransmissions and a receive window pinned at zero — which is a conclusive proof that the network delivered everything and the application stopped reading. Proving it in the first minute changes who owns the incident.

### 2. Symptom fingerprints: choosing a branch before running anything

The shape of a symptom carries more information than its magnitude. Six shapes cover most incidents; each one is a *mechanism signature*, not a heuristic, and the "why" column is what makes it usable on a shape you have not seen before.

| Shape of the symptom | Why that shape happens | Branch |
|---|---|---|
| Latency **quantised** at a fixed value or its multiples (5 s, 10 s, 30 s) | a **timer** fired. Nothing else produces a repeated identical delay: variable load produces a distribution, a timeout produces a spike at exactly one value | resolver retry (5 s), TCP SYN retry (1, 3, 7 s), client timeout — check which timer equals the number |
| **Size-dependent**: small requests fine, large ones hang (not reset) | the handshake and small segments fit the path; the first full-size segment does not, and the ICMP that would say so is being dropped | path MTU / PMTUD black hole |
| **Load-dependent**: fine at low rate, fails above a threshold | a **table or queue** filled. Tables fail at a ceiling, links degrade gradually | conntrack, ephemeral ports, backlog, qdisc, NIC ring |
| **One-sided**: A→B fails, B→A fine; or fails from one node only | asymmetric state — policy, routing, NAT, or one node's dataplane program | NetworkPolicy, node-local dataplane, asymmetric routing |
| **Starts at a deploy or scale-up** | the new thing has different labels, a different image, a different instance type, or a different MTU | identity/policy mismatch, MTU heterogeneity, missing DaemonSet coverage on the new pool |
| **Slow with zero errors anywhere** | nothing is failing; something is *choosing* a slower path | offload/GDR fallback (07), congestion control, app stall, mesh added to the path |

Two of those deserve extra emphasis because they are the ones that most often get misread.

**Quantisation is the strongest signal in networking.** If your p99 latency histogram has a spike at 5.00 s and nothing between 200 ms and 5 s, you are not looking at congestion, a slow backend, or a busy node — all of those produce a *smear*. You are looking at something that gave up and waited for a timer. The follow-up question is only "which timer is 5 seconds", and the answer on a Kubernetes cluster is almost always glibc's resolver `RES_TIMEOUT` (mechanism in 02 and 01b.7). **Match the number to a timer before you form any other theory.**

**"Slow with zero errors" is the shape that defines GPU clusters** and is the reason lesson 07 exists. Every other shape has *something* failing. This one has everything succeeding, more slowly, because a piece of software chose a fallback path it is designed to choose: NCCL staging through host DRAM, TCP falling back from a large-send offload, a mesh sidecar being added to a path, or a congestion-control algorithm exiting slow start late. **When nothing is broken and everything is slower, look for a fallback, not a fault.**

### 3. One request, every layer, and which tool sees which hop

Before the tree, the map. Here is a single HTTP request from a pod in `serving` to a Service backed by pods in `feature-store`, on a cluster with an eBPF dataplane and a sidecar mesh, with every observation point marked.

```
  ONE REQUEST, AND WHO CAN SEE IT
  ═══════════════════════════════════════════════════════════════════════════

  CLIENT POD (serving/api-7f9)                 tool that sees this hop
  ────────────────────────────                 ───────────────────────
   ① app calls connect()/write()               strace, app tracing (OTel)
        │                                       ss -tie  (socket state)
   ② socket + send queue                       ss -tim  (skmem, notsent)
        │                                       ← THE CHEAPEST TRUTH
   ③ cgroup eBPF at connect()                  cilium-dbg service list
        │  socket-LB rewrites the VIP           (no packet exists yet;
        │  BEFORE a packet exists                no conntrack entry either)
        ▼
   ④ sidecar interception (iptables            istioctl proxy-config
        │  REDIRECT → Envoy :15001)             envoy access log + stats
        │  ── mesh adds a full L7 hop ──        RESPONSE FLAGS (06 §7)
        ▼
   ⑤ veth → host netns                          pwru (per-hook, per-skb)
        │                                        kfree_skb tracepoint
   ⑥ tc/eBPF ingress+egress programs            hubble observe (verdict,
        │  policy decision on IDENTITY           identity, drop reason)
        │  NAT / conntrack if netfilter          conntrack -L, conntrack -S
        ▼                                        (01b.7)
   ⑦ routing + qdisc + NIC                       tc -s qdisc, ethtool -S,
        │                                        /proc/net/softnet_stat
        ▼                                        (A02.1 §5)
  ═════ THE WIRE ═════════════════════════ VPC flow logs (sampled, delayed,
        │                                        IP-keyed, minutes late)
        ▼                                        cloud LB metrics (opaque)
  SERVER NODE — mirror image of ⑦ back up to ①
        │
        ▼
   ⑧ server app read()                          ss -tie ON THE SERVER
                                                 (rcv_space, rwnd, skmem r/rb)

  ─────────────────────────────────────────────────────────────────────────
  AND THE PATH THAT APPEARS NOWHERE ABOVE:

   GPU HBM ──▶ HCA ──▶ 400G wire        no socket   → ss sees nothing
                                         no skb      → pwru sees nothing
                                         no tc hook  → hubble sees nothing
                                         no proxy    → Envoy sees nothing
              observable ONLY via: ethtool -S <pf> (per-priority counters),
              ibstat / perfquery, NCCL_DEBUG=INFO, nccl-tests busbw  (07)
```

**Three consequences of that picture, which are the reason to draw it.**

First, **the tools are not interchangeable and their disagreements are informative, not noise.** An empty `conntrack -L` on a socket-LB cluster means "no entry exists by design" (lesson 05 §7), not "no traffic". A Hubble flow marked `FORWARDED` and an application timeout are not a contradiction: the dataplane delivered the packet and something above it did not act on it.

Second, **the cheapest observation point is the one closest to the application**, and it is at both ends. Two `ss -tie` invocations — one client-side, one server-side — bracket the entire network. If both look healthy, the network is not your problem, and you have proven it with two commands and no privileges beyond the pods themselves.

Third, **the mesh hop is invisible to every packet-level tool.** Envoy terminates the connection; the socket the client sees ends at the sidecar. So a client-side `ss` on a meshed pod measures the RTT to a proxy on the same machine — microseconds — and tells you nothing about the real upstream. On a meshed path the authoritative source is the proxy's own telemetry.

### 4. Reading a socket: `ss -tie`, field by field

This is the instrument to learn cold, because it is the only one that distinguishes the four causes of "slow" without any cluster tooling at all. The field names and their order below are from iproute2's `tcp_stats_print()` in `misc/ss.c`, which is the authority — several of the most useful fields are not in the man page.

```console
$ kubectl exec -n serving api-7f9 -- ss -tiem 'dst 10.244.9.0/24'
ESTAB 0  1868608  10.244.3.7:52488  10.244.9.21:8080
     skmem:(r0,rb131072,t0,tb4194304,f4096,w1871872,o0,bl0,d0)
     cubic wscale:7,7 rto:204 rtt:3.412/0.688 ato:40 mss:1398 pmtu:1450
     rcvmss:536 advmss:1448 cwnd:512 ssthresh:64 bytes_sent:4831838208
     bytes_acked:4831834944 bytes_received:8842 segs_out:3455612
     segs_in:1210044 data_segs_out:3455602 send 1678000000bps
     lastsnd:4 lastrcv:88 lastack:4 pacing_rate 2013600000bps
     delivery_rate 1244000000bps delivered:3454272 app_limited
     busy:214880ms rwnd_limited:198340ms(92.3%) unacked:2
     retrans:0/0 rcv_space:14480 rcv_ssthresh:64088 notsent:1868608
     minrtt:2.881
```

*(Representative rather than captured: every field name, its order and its unit are from `tcp_stats_print()`; the values are constructed and, as the read-through below shows, arithmetically self-consistent.)*

Read it in this order, because the order is a decision procedure:

- **`retrans:0/0`** — current/total retransmissions. **Zero total is a proof, not an indication:** across 3.4 million data segments, not one needed resending. *There is no packet loss on this path.* That single field eliminates the entire loss branch — NIC, qdisc, fabric, congestion drops — in one glance. When it is non-zero, compare `bytes_retrans` to `bytes_sent` for the rate rather than reading the raw count.
- **`rtt:3.412/0.688` and `minrtt:2.881`** — smoothed RTT and its mean deviation, in **milliseconds**, and the minimum ever observed. The gap between `rtt` and `minrtt` is queueing: 3.4 vs 2.9 ms is 0.5 ms of standing queue, which is nothing. A `rtt` several times `minrtt` with `retrans:0` is the bufferbloat signature — the packets arrive, they just wait somewhere.
- **`cwnd:512` with `ssthresh:64`** — the congestion window is far above the slow-start threshold, so the sender is in congestion avoidance and has been for a long time without a loss event. The sender is not being throttled by congestion control.
- **`rwnd_limited:198340ms(92.3%)` out of `busy:214880ms`** — **this is the answer.** `busy` is how long the socket has had data to send; `rwnd_limited` is how much of that time it was blocked *by the receiver's advertised window*. 92.3% of the time this connection wanted to send and was not allowed to, because the peer's window was closed. Its siblings: `sndbuf_limited` (blocked by our own send buffer — raise `SO_SNDBUF` or `net.ipv4.tcp_wmem`) and `app_limited` (we simply had nothing to send). **These three fields partition "why was this connection not going faster" exhaustively, and none of them is documented in `ss(8)` — they exist only in the source.**
- **`notsent:1868608` with `unacked:2`** — 1.8 MB written by the application and sitting in the send queue, unsent, while only **two segments** are actually in flight. Do the arithmetic across the fields, because it is the proof: `bytes_sent − bytes_acked = 4831838208 − 4831834944 = 3,264 B`, which is those two segments; `skmem`'s `w1871872` (`wmem_queued`) minus `notsent` leaves the same 3,264 B; and the queue is 1.8 MB of a `tb4194304` (4 MB) send buffer. **A sender with `cwnd:512` and two packets in flight is not congestion-limited by anything on its own side** — it is being held by the receiver's window, exactly as `rwnd_limited` says.
- **`skmem:(r0,rb131072,t0,tb4194304,f4096,w1871872,o0,bl0,d0)`** — from `print_skmeminfo()`: `r` = receive-queue bytes allocated, `rb` = receive buffer size, `t` = transmit allocated, `tb` = send buffer size, `f` = forward-allocated, `w` = write-queue bytes, `o` = option memory, `bl` = backlog, **`d` = drops**. `d0` means this socket has dropped nothing itself. A non-zero `d` on a *receiving* socket, together with `r` pinned at `rb`, is the "application is not calling `read()` fast enough" signature.
- **`pacing_rate` / `delivery_rate` / `send`** — `send` is the instantaneous cwnd/RTT estimate, `delivery_rate` is what was actually achieved recently, `pacing_rate` is the ceiling the pacer imposes. `delivery_rate` well below `send` with `app_limited` set means the *sender* was the limit and the estimate is not meaningful.
- **`pmtu:1450` vs `mss:1398` vs `advmss:1448`** — the path MTU the socket believes in, the MSS in use, and the MSS this side advertised. A `pmtu` that equals the interface MTU while large transfers hang is the PMTUD black hole from lesson 05 §5; a `pmtu` that has *dropped* below the interface MTU means PMTUD worked.

**The verdict from this transcript, stated the way you would state it in an incident channel:** *"Client socket to feature-store: 3.4 M segments sent, zero retransmissions, RTT 3.4 ms against a 2.9 ms minimum, and the connection was receiver-window-blocked for 92% of its busy time with 1.8 MB stuck in the send queue. The network delivered everything we gave it. This is a consumer that stopped reading."* That is a five-second command producing a conclusion that reassigns the incident.

### 5. Where the counters end and the reasons begin

A02.1 §5 gives the table that maps each datapath stage to its counter: `ethtool -S` for the NIC ring, `/proc/net/softnet_stat` for softirq and backlog, `tc -s qdisc` for the qdisc, `nstat` for the protocol layers. That table answers **"at which stage did packets die?"** and it is still the right first move for a host-level problem.

What it cannot answer is **"why"**. `rx_dropped` incrementing tells you a stage; it does not tell you whether the packet was malformed, filtered, unroutable, or simply late. Historically you closed that gap by reading kernel source around the counter. You no longer have to.

### 6. The kernel's drop taxonomy

The `skb:kfree_skb` tracepoint carries a **drop reason** — an enum the kernel sets at every discard site. From `include/trace/events/skb.h`, the tracepoint's signature and print format are:

```c
TRACE_EVENT(kfree_skb,
    TP_PROTO(struct sk_buff *skb, void *location,
             enum skb_drop_reason reason, const struct sock *rx_sk),
    ...
    TP_printk("skbaddr=%p rx_sk=%p protocol=%u location=%pS reason: %s", ...)
);
```

Note what is in there: the **location** (`%pS` — the kernel function and offset that freed it) and the **reason**, symbolically. The enum is defined in `include/net/dropreason-core.h` and currently carries well over 200 values. The ones that matter for a Kubernetes bisection map cleanly onto the branches of §1:

| Reason | Means | Branch it confirms |
|---|---|---|
| `NETFILTER_DROP` | dropped by a netfilter hook | iptables / NetworkPolicy / conntrack (05, 01b.7) |
| `BPF_CGROUP_EGRESS`, `TC_INGRESS`, `TC_EGRESS`, `XDP` | dropped by an eBPF program at that hook | CNI dataplane — go to Hubble for *which policy* |
| `SOCKET_RCVBUFF` | receive buffer full | the application is not reading — §4's branch |
| `SOCKET_BACKLOG` | socket backlog full while the socket was locked | receiver-side, still an app-speed problem |
| `TCP_LISTEN_OVERFLOW` | listener accept queue full | app not calling `accept()` fast enough |
| `CPU_BACKLOG` | per-CPU backlog full (`netdev_max_backlog`) or RPS flow limit | softirq/steering — A02.1's R3 stage |
| `QDISC_DROP` | dropped by a qdisc on enqueue/dequeue | qdisc/AQM — A02.1's T4 stage |
| `PKT_TOO_BIG` | exceeded an egress MTU | the MTU branch, from the kernel's own mouth |
| `IP_OUTNOROUTES`, `IP_INNOROUTES` | no route | routing, not policy |
| `NEIGH_FAILED`, `NEIGH_QUEUEFULL` | ARP/ND resolution failed or its queue filled | L2 — a genuinely different problem |
| `FRAG_REASM_TIMEOUT`, `DUP_FRAG` | fragment reassembly failed | fragmentation — often downstream of an MTU problem |
| `TCP_RFC7323_PAWS` | timestamp check failed (old/duplicate) | asymmetric routing, or a NAT reusing tuples too fast |
| `NO_SOCKET` | no listener for this tuple | the process is gone, or you are on the wrong node |

The cheapest way to read it, on any modern kernel and with no extra tooling installed:

```console
# echo 1 > /sys/kernel/tracing/events/skb/kfree_skb/enable
# cat /sys/kernel/tracing/trace_pipe
  curl-31844 [012] ..s1. 91827.442013: kfree_skb: skbaddr=0xffff9a3c41d8e300
      rx_sk=0x0 protocol=2048 location=nf_hook_slow+0x8f/0x170
      reason: NETFILTER_DROP
# echo 0 > /sys/kernel/tracing/events/skb/kfree_skb/enable
```

*(Representative: the print format and the field order are exact, from the tracepoint definition; the addresses and timings are illustrative.)*

Two lines of that output do most of the work. `reason: NETFILTER_DROP` says which *subsystem* owns the drop, which is the branch. `location=nf_hook_slow+0x8f` says which *function* did it, which is the leaf. **This is the single highest-value-per-keystroke command in host-level network debugging**, and it needs nothing installed.

Its limits are real, and you should state them before someone else does: the tracepoint fires for every dropped skb on the box, so on a busy node you need a filter (`echo 'reason != 0' > .../kfree_skb/filter`, or a `bpftrace` one-liner aggregating by reason) or you will drown; it tells you nothing about packets that were *delivered* to the wrong place; and `NOT_SPECIFIED` is still a common reason at older or vendor drop sites, which means "the kernel dropped this and nobody has annotated why yet".

Purpose-built drop monitors — `dropwatch`, and Red Hat's `retis` — sit on the same tracepoints and add aggregation, symbol resolution and filtering. They are a convenience layer over this mechanism, not a different one, which is why understanding the mechanism transfers to whichever tool your fleet has.

### 7. Hubble: the flow record, and what "identity" buys

The eBPF dataplane sees every packet at the tc/cgroup hooks and, crucially, it knows *what each endpoint is*. Cilium assigns a cluster-wide numeric **security identity** to each unique set of security-relevant labels (lesson 05 §10); Hubble exports flows keyed on those identities rather than on addresses.

A flow record is a protobuf, and knowing its fields tells you exactly what questions it can answer. From `cilium/cilium`'s `api/v1/flow/flow.proto`:

- **`verdict`** — one of `FORWARDED`, `DROPPED`, `ERROR`, `AUDIT` (would have been dropped, but policy audit mode is on), `REDIRECTED` (sent to a proxy), `TRACED` (observed at a trace point, no verdict yet), `TRANSLATED` (an address was rewritten).
- **`drop_reason_desc`** — Cilium's own drop enum, distinct from the kernel's: `POLICY_DENIED`, `POLICY_DENY`, `CT_MAP_INSERTION_FAILED`, `STALE_OR_UNROUTABLE_IP`, `NO_TUNNEL_OR_ENCAPSULATION_ENDPOINT`, `IP_FRAGMENTATION_NOT_SUPPORTED`, `AUTH_REQUIRED`, `INVALID_IDENTITY`, `TTL_EXCEEDED`, `UNENCRYPTED_TRAFFIC`, and others.
- **`egress_denied_by` / `ingress_denied_by` / `egress_allowed_by` / `ingress_allowed_by`** — repeated `Policy` messages, each carrying `name`, `namespace`, `labels`, `revision` and `kind`. **This is policy correlation: the flow record names the actual object that made the decision.**
- **`is_reply`**, `auth_type`, and the L7 payload for HTTP/DNS/Kafka when L7 visibility is enabled.

In use it looks like this. The output formats below are from Cilium's own documentation and the Hubble README:

```console
$ hubble observe --pod deathstar --verdict DROPPED
May  4 13:23:43.791: default/tiefighter:42742 -> default/deathstar-c74d84667-cx5kp:80
    http-request DROPPED (HTTP/1.1 PUT http://deathstar.default.svc.cluster.local/v1/exhaust-port)
May  4 13:23:47.852: default/xwing:42818 <> default/deathstar-c74d84667-cx5kp:80
    Policy denied DROPPED (TCP Flags: SYN)
```

Read the two lines as different findings. The first is an **L7** drop: the connection was allowed, the HTTP method was not. The second is an **L3/L4** drop at the SYN — the connection never formed. Same verdict, entirely different fix, and the distinction is visible in the flow type, not the verdict.

The `<>` versus `->` in those lines matters too: `->` is a flow whose direction is known, `<>` is one where it is not (typically a drop before the datapath established a direction).

Three things Hubble does that IP-based telemetry structurally cannot:

- **Survive churn.** `default/xwing` is a stable name across restarts, redeploys and IP reassignment. A flow log entry naming `10.0.3.7` refers to whatever holds that address *now*.
- **Answer "which policy".** With policy correlation enabled, a denied flow carries the denying `CiliumNetworkPolicy`'s name and namespace, which turns "something denied this" into "edit this object". Without it you are diffing policies by hand against label sets.
- **Filter on identity, not topology.** `--label`, `--namespace`, `--to-fqdn`, and raw `--allowlist`/`--denylist` filters (JSON-encoded `FlowFilter` messages) select on what a workload *is*.

And three honest limits, which belong in the runbook next to the commands:

- **Scope.** The Hubble API on a Cilium agent is **node-local by default**. Cluster-wide observation requires Hubble Relay; without it, `hubble observe` on the wrong node shows you nothing and looks exactly like "no traffic".
- **Retention.** Hubble keeps a bounded in-memory ring buffer per node. It is a live and recent-past tool. Compliance-grade retention is what flow logs are for.
- **A `DROPPED` verdict is not a bug.** A default-deny NetworkPolicy denying traffic exactly as designed produces a continuous stream of `Policy denied DROPPED`. The tool reports what happened and why; whether it *should* have happened is still your judgement.

**Where flow logs still win**, and this is a cross-team point as much as a technical one: cloud VPC flow logs are sampled and delayed by minutes, but they are retained for months, they cover traffic Cilium never sees (out-of-cluster, cross-account, load-balancer front ends), and they are usually owned by a security or cloud team on a forensics timescale. Hubble replaces reaching for them *first*; it does not replace them.

### 8. `pwru`: tracing one packet through every hook

When the flow layer says "it left" and the far side says "it never arrived", you need per-hook visibility. `pwru` attaches to a large set of kernel networking functions and prints every one that a matching skb traverses, so you see the exact function where it stopped.

The invocation is a pcap-filter expression plus flags (from the tool's own `--help`):

```console
# pwru --output-tuple --output-meta --timestamp relative \
       'host 10.244.9.21 and port 8080'
SKB                CPU PROCESS          TIMESTAMP        NETNS      MARK/x
0xffff9a3c41d8e300 12  curl             1041             4026533112 0
      IFACE     PROTO MTU   LEN   TUPLE                              FUNC
      eth0:3    0x0800 1450  1500  10.244.3.7:52488->10.244.9.21:8080 ip_output
0xffff9a3c41d8e300 12  curl             1078             ...                nf_hook_slow
0xffff9a3c41d8e300 12  curl             1102             ...                kfree_skb_reason
```

*(Representative: the column layout and flag names are exact, the addresses and function sequence are illustrative.)* The column layout above (`SKB`, `CPU`, `PROCESS`, `TIMESTAMP`, `NETNS`, `MARK/x`, `IFACE`, `PROTO`, `MTU`, `LEN`, `TUPLE`, `FUNC`) is exactly what `PrintHeader()` in `internal/pwru/output.go` emits. **The `SKB` column is the identity that makes the output readable**: the same pointer appearing on consecutive lines is one packet moving through the stack, and the last function before it disappears is where it died.

The flags that matter operationally:

| Flag | What it does | When you need it |
|---|---|---|
| `--filter-track-skb` | keep tracing a packet even after it stops matching your filter | **NAT and encapsulation** — the tuple changes mid-path |
| `--filter-track-skb-by-stackid` | keep tracing across `kfree_skb` (e.g. through a bridge) | packets that are freed and re-created |
| `--filter-tunnel-pcap-l2` / `-l3` | a second pcap expression matched against the *inner* header of a VXLAN/Geneve packet | overlay clusters |
| `--filter-netns` / `--filter-ifname` | scope to one namespace or interface | reduces cost enormously on a busy node |
| `--filter-func` | restrict probes to functions matching an RE2 regex | reduces cost enormously |
| `--filter-trace-tc` / `--filter-trace-xdp` | also trace tc and XDP BPF programs | eBPF dataplanes |
| `--output-caller` / `--output-stack` | print the calling function / full stack | when the function name alone is ambiguous |
| `--backend kprobe-multi` | attach with fprobe instead of individual kprobes | much cheaper attach; needs kernel ≥ 5.18 |

**A correction worth carrying**, because the widely-cited version of this story is out of date: the classic `pwru` limitation was that a 5-tuple filter stops matching once a packet is encapsulated or NATed, forcing you to correlate manually by skb pointer across hooks. That is exactly what `--filter-track-skb` and the `--filter-tunnel-pcap-l2/l3` expressions exist for, and they are in the tool's flag set today. The manual skb-pointer correlation is the fallback, not the procedure.

**The cost is real and is the reason this is step 4, not step 1.** `pwru` requires ≥ 5.3 (≥ 5.9 for `--output-skb`, ≥ 5.18 for `kprobe-multi`), `CONFIG_DEBUG_INFO_BTF`, usually a privileged pod with `/sys/kernel/debug` mounted and `hostNetwork`/`hostPID`, and kprobe-based tracing at high packet rates has non-trivial CPU cost. Standard practice is to **reproduce in staging, or scope the filter as narrowly as the reproduction allows** — by netns, by interface, by function regex — rather than running it broadly against live production traffic.

### 9. Capture, and what it is still for

You cannot tcpdump a fleet, and everything above exists so that you rarely need to. But capture remains the only tool that shows you *bytes on a wire*, and there are questions only bytes answer: what exactly is in this TLS handshake, does the peer's HTTP/2 SETTINGS frame say what we think, is this a real RST or a proxy synthesising one.

Two disciplines make capture usable at scale:

**Scope before you capture.** `tcpdump -i any -n -c 200 'host 10.244.9.21 and tcp port 8080 and (tcp[tcpflags] & (tcp-syn|tcp-rst|tcp-fin) != 0)'` captures only connection-state changes — a few hundred packets that answer "who is closing this connection and when" without a gigabyte of payload. Bounded capture (`-c`), bounded snaplen (`-s`), and a filter that encodes the hypothesis are what separate targeted capture from data hoarding.

**Know that the host lies about segmentation.** With GRO and TSO enabled, a host-side capture is taken *above* the offload, so you see 64 KB "segments" that were never on the wire, and you cannot see real fragmentation or the true on-wire MTU (A02.1 §12 has the mechanism). For any question about MTU or segmentation: capture on the wire (SPAN/tap), or disable the offload for the duration of the test (`ethtool -K eth0 gro off tso off gso off`), and remember to turn it back on.

**Reading a capture for the three signatures you actually look for:**

```
14:22:01.104  10.244.3.7.52488 > 10.244.9.21.8080: Flags [P.], seq 1:1449, length 1448
14:22:01.304  10.244.3.7.52488 > 10.244.9.21.8080: Flags [P.], seq 1:1449, length 1448
14:22:01.712  10.244.3.7.52488 > 10.244.9.21.8080: Flags [P.], seq 1:1449, length 1448
   → the SAME sequence number, at ~200 ms then ~400 ms: RTO retransmission with
     exponential backoff. Nothing acked it. Combined with a successful handshake
     and a small MTU downstream, this is the PMTUD black hole (05 §5).

14:22:03.001  10.244.9.21.8080 > 10.244.3.7.52488: Flags [.], ack 4096, win 0
   → win 0: the receiver advertised a CLOSED window. The peer's application
     stopped reading. This is §4's rwnd_limited, seen from the outside.

14:22:04.550  10.244.9.21.8080 > 10.244.3.7.52488: Flags [R], seq 4096
   → a reset. Ask who sent it: a real backend, a proxy on either side, or a
     middlebox. On a meshed path (§3 ④) the client's peer is a local sidecar,
     so a client-side RST says nothing about the real upstream.
```

### 10. Where this toolkit's vision ends

Three boundaries, all worth naming explicitly rather than discovering mid-incident.

**The mesh.** Envoy terminates connections. Packet-level tools see the client↔sidecar segment and the sidecar↔upstream segment as unrelated connections, and the interesting state — retry counts, circuit-breaker trips, outlier ejections, upstream selection — exists only inside the proxy. The authoritative signal is the proxy's **response flags** (lesson 06 §7: `UO` circuit breaker, `UF` connection failure, `UH` no healthy upstream, `URX` retry limit, `NR` no route, `UT` upstream timeout, `DC` downstream disconnect, `RL` rate limited) and `istioctl proxy-status`'s `STALE` marker, which says a proxy's endpoint list is out of date. **A 503 with `UO` is not a network event at all**; treating it as one costs an hour.

**The cloud load balancer and everything upstream of the VPC.** You get metrics and sampled flow logs, on the provider's schedule and in the provider's vocabulary. Lesson 03's mechanisms (health-check cascades, panic mode, connection draining) have to be diagnosed from those aggregates plus your own backends' view. Plan the correlation key in advance — a request ID injected at the edge and logged at every tier — because you cannot add it during an incident.

**The RDMA fabric — a hard boundary, not a gap.** Hubble instruments tc/cgroup hooks. `pwru` instruments skb-handling functions. `ss` reads socket state. **RDMA traffic creates no skb, traverses no tc hook, and has no socket**, because deleting exactly those stages is what RDMA is for (07, 09.3). Pointing these tools at a slow all-reduce produces silence, and *the silence is itself diagnostic*: if the application says the collective is slow and the entire kernel-level toolkit shows nothing, you are on the fabric branch and should be reading lesson 07's signals instead — `NCCL_DEBUG=INFO`'s `GPU Direct RDMA Enabled/Disabled` lines, `nccl-tests` busbw against the acceptance baseline, and `ethtool -S <pf> | grep prio` for pause frames, ECN marks, and the discard counters that must be zero.

### 11. The decision tree, assembled

Everything above collapses into the artefact. This is the deliverable; the sections above are why each branch is where it is.

```
  SYMPTOM ARRIVES
  ═════════════════════════════════════════════════════════════════════════
  0. CLASSIFY THE SHAPE (§2)  — 0 commands
     quantised? size-dependent? load-dependent? one-sided? post-deploy?
     slow-with-no-errors?  → sets the prior; does NOT skip the tests.
        │
        ▼
  1. IS IT THE NETWORK AT ALL?          ss -tie  (client, then server)
        │
        ├── retrans:0/0, rtt≈minrtt, and rwnd_limited / sndbuf_limited /
        │   app_limited dominant, or skmem d>0 on the receiver
        │      ⇒ NOT THE NETWORK. Hand to the app owner with the numbers.
        │        (§4 — this ends a large fraction of incidents.)
        │
        ├── retrans climbing, rtt≈minrtt        ⇒ LOSS. go to 4.
        ├── rtt ≫ minrtt, retrans 0             ⇒ QUEUEING. tc -s qdisc,
        │                                          then A02.1 §5's table.
        └── socket never reaches ESTAB          ⇒ go to 2.

  2. IS IT NAME RESOLUTION?             timed lookup loop + resolv.conf
        │  fingerprint: delays quantised at 5 s multiples
        ├── yes  ⇒ ndots search-list walk, or the parallel-A/AAAA conntrack
        │          insert race. Mechanism: 02, 01b.7. Fixes: NodeLocal
        │          DNSCache, single-request-reopen, ndots:1, TCP.
        └── no, delays are not quantised  ⇒ go to 3.

  3. IS IT PATH MTU?                    ping -M do sweep; ss's pmtu vs mss
        │  fingerprint: handshake + small requests fine, first full-size
        │  segment hangs (hangs, not resets)
        ├── yes  ⇒ overlay MTU vs underlay, heterogeneous node pools, and
        │          check that ICMP frag-needed is not being policy-dropped.
        │          Mechanism and the three fixes: 05 §5.
        └── no   ⇒ go to 4.

  4. IS THE DATAPLANE DROPPING IT?
        │  a) cheapest, no tooling:   kfree_skb tracepoint → reason  (§6)
        │  b) cluster-wide, identity: hubble observe --verdict DROPPED (§7)
        │
        ├── reason NETFILTER_DROP / verdict DROPPED "Policy denied"
        │      ⇒ policy. Read egress_denied_by / ingress_denied_by for the
        │        object name. Check whether the drop is CORRECT (§7).
        ├── reason CPU_BACKLOG / QDISC_DROP / SOCKET_RCVBUFF
        │      ⇒ host resource, not a policy. A02.1 §5's stage table.
        ├── conntrack drop / insert_failed climbing (conntrack -S)
        │      ⇒ table full, or an insertion race. 01b.7.
        ├── nothing dropped anywhere, but the far side never received it
        │      ⇒ pwru, scoped by netns + function regex, with
        │        --filter-track-skb if NAT or encapsulation is in play (§8)
        └── everything forwarded, verdicts clean  ⇒ go to 5.

  5. IS IT THE LB OR THE MESH?          proxy stats, not app logs  (§10)
        │
        ├── Envoy response flag UO/UF/UH/URX/NR/UT ⇒ proxy-level cause,
        │      with a specific mechanism per flag (06 §7–§9)
        ├── istioctl proxy-status shows STALE     ⇒ the dataplane's endpoint
        │      list is out of date; a control-plane problem (06 §6)
        └── proxies clean, endpoints fresh        ⇒ go to 6.

  6. IS IT THE FABRIC?                  ← THIS LESSON'S TOOLS CANNOT SEE IT
        Tell: the collective/job is slow, and steps 1–5 are ALL clean and
        silent. Silence here is evidence, not absence of evidence (§10).
        Hand off to 07:  NCCL_DEBUG=INFO  |  nccl-tests busbw vs baseline
                         ethtool -S <pf> | grep prio3   (discards MUST be 0)
```

**The property that makes this a tree and not a checklist is that every branch terminates**, either in a cause or in an explicit hand-off to a named owner with a named artefact. A runbook step whose outcome is "hmm, looks fine" is a bug in the runbook.

## Perspectives

**Procedural.** This is differential diagnosis, and it is not a networking technique — it is the same discipline that debugs a build pipeline, a data-corruption bug, or a flaky test. Each branch has a cheap test with a trustworthy negative; the branches are ordered by cost over discriminating power; and you rule out rather than confirm, because confirmation bias is what turns a bisection into a random walk. When an interviewer asks "how would you debug X", the ordering instinct is what is being scored, not the tool list.

**Tooling-lifecycle.** Every tool here is running code with a cost and a blast radius. `pwru`'s kprobe attachment is measurable CPU at high packet rates; enabling `kfree_skb` tracing on a busy node without a filter floods the trace buffer; L7 visibility in a mesh or in Hubble multiplies telemetry cardinality. **Knowing when *not* to reach for full-fidelity tracing is as much a staff skill as knowing it exists** — and the discipline that makes it unnecessary is having run steps 0–3 first.

**Cross-team.** Identity-based telemetry (Hubble, mesh SPIFFE IDs) and IP-based telemetry (VPC flow logs) are owned by different teams on different timescales: the platform team's live bisection tool versus the security or cloud team's months-retained audit artefact. Knowing which artefact exists, who can query it, and how long it takes to get an answer is what keeps an incident from stalling on a ticket. Decide in advance which questions you will answer live and which you will answer next week.

**GPU-fleet.** The fabric branch is the one place the toolkit genuinely stops, and the correct professional response is to say so and hand off, not to keep pointing kernel tools at traffic that never enters the kernel. The transferable idea is more general: **know the edge of your instrumentation, and treat a tool's silence as a data point only when you know what that tool can see.** Silence from Hubble on an RDMA problem is expected; silence from Hubble on an HTTP problem means Hubble is not running on the node you are asking.

## Real-world use cases

- **Cloudflare, "Lost in transit: debugging dropped packets from negative header lengths."** A production incident traced with `pwru`: packets vanished after IPVS native encapsulation, and the trace located the exact kernel function where the skb was freed, leading to an upstream Linux fix. What it shows: per-hook tracing answers a question no counter can, because a counter reports a *stage* and the bug was a *function*. It is also the origin of the widely-repeated encapsulation-filter caveat — which, as §8 notes, `--filter-track-skb` and the tunnel-pcap expressions now address directly. https://blog.cloudflare.com/lost-in-transit-debugging-dropped-packets-from-negative-header-lengths/
- **Mark Betz, "Exhausting conntrack table space crippled our k8s cluster."** A bisection narrative that follows §11's shape exactly: load-dependent symptom → `dmesg` showing `nf_conntrack: table full, dropping packet` → count against the limit → fix. What it shows: the load-dependent fingerprint (§2) pointing at a *table* rather than a link, and the value of a symptom shape that is diagnostic before any tool is opened. https://medium.com/@betz.mark/exhausting-conntrack-table-space-crippled-our-k8s-cluster-98564f6f34e0
- **Datadog, "It's always DNS . . . except when it's not."** An engineering team walking the DNS branch, then the conntrack branch, then the client/LB branch, and converging on a non-obvious root cause rather than stopping at the first plausible one. What it shows: the discipline of *continuing to rule out* after finding a contributing factor — the most common way a real bisection goes wrong is stopping one branch early. https://www.datadoghq.com/blog/engineering/grpc-dns-and-load-balancing-incident/

## Worked example

**Scenario.** The `serving` namespace pages: p99 latency to `feature-store` went from 40 ms to 2.8 s starting Tuesday 09:10, affecting roughly 12% of requests. A new node pool was added Monday evening. Cilium is the dataplane; Istio meshes the frontend but not `feature-store`. Bisect it.

**Step 0 — classify the shape, before touching anything (§2).**

```
   p99 2.8 s, p50 unchanged at 41 ms, no quantisation anywhere in the
   histogram (no spike at 1 s, 3 s, 5 s, 10 s).
   Failure ratio 12 %, stable, not growing with load.
   Started at a scale-up, not at a deploy.
   Errors: none. Every affected request eventually SUCCEEDS.
```

Read that hard. **No quantisation rules out every timer-driven branch**, which includes the resolver retry (would spike at 5 s) and TCP SYN retries (1 s, 3 s, 7 s). **"Slow but succeeds" rules out drops**, at least for the affected requests, because a dropped-and-not-retried request fails. **"12%, stable, not load-correlated" smells like a subset of endpoints**, because a resource ceiling would grow with load and a global misconfiguration would affect 100%. Prior: something about a subset of paths is slower. Do not skip the tests.

**Step 1 — the client socket (§4).** Run it on a pod that is seeing the latency, filtered to the affected destination range.

```console
$ kubectl exec -n serving api-7f9 -- ss -tiem 'dst 10.244.9.0/24'
ESTAB 0 1868608 10.244.3.7:52488 10.244.9.21:8080
     skmem:(r0,rb131072,t0,tb4194304,f4096,w1871872,o0,bl0,d0)
     cubic wscale:7,7 rto:204 rtt:3.412/0.688 mss:1398 pmtu:1450
     cwnd:512 ssthresh:64 bytes_sent:4831838208 bytes_acked:4831834944
     segs_out:3455612 data_segs_out:3455602 retrans:0/0
     busy:214880ms rwnd_limited:198340ms(92.3%) unacked:2
     notsent:1868608 minrtt:2.881
```

**Four facts, one conclusion.** `retrans:0/0` over 3.4 M segments: no loss on this path, so the whole loss branch (steps 3–4) is dead. `rtt:3.412` against `minrtt:2.881`: 0.5 ms of queueing, i.e. none. `rwnd_limited:198340ms(92.3%)`: this connection spent 92% of its busy time blocked by the **receiver's** advertised window. `notsent:1868608`: 1.8 MB written by our app and stuck in our send queue because the peer will not accept it.

**The network delivered every byte it was given. The receiver stopped reading.** That should have been the end of it — except the symptom is *latency*, not throughput, and 12% is a suspiciously specific fraction. So confirm the negative and find out *which* receivers.

**Step 2 — rule out DNS anyway, because it costs one command and the negative is cheap.**

```console
$ kubectl exec -n serving api-7f9 -- sh -c \
    'for i in $(seq 200); do
       /usr/bin/time -f %e getent hosts feature-store.feature-store.svc.cluster.local
     done 2>&1 | awk "{print \$1}" | sort -n | tail -3'
0.004
0.004
0.006
```

Max 6 ms across 200 lookups, no 5-second outliers. **DNS branch closed**, consistent with the shape. (Had this shown 5.00 s entries, the mechanism and the three fixes are lesson 02's and 01b.7's.)

**Step 3 — rule out MTU, one command.** The new node pool is a different instance type, which is the classic source of heterogeneous underlay MTU.

```console
$ kubectl exec -n serving api-7f9 -- ping -M do -s 1422 -c 2 10.244.9.21
PING 10.244.9.21 (10.244.9.21) 1422(1450) bytes of data.
1430 bytes from 10.244.9.21: icmp_seq=1 ttl=63 time=0.412 ms
$ kubectl exec -n serving api-7f9 -- ip link show eth0 | head -1
3: eth0@if17511: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1450
$ for n in gpu-a3-011 gpu-c4-101; do
    kubectl debug node/$n -q --image=busybox -- ip link show eth0 | awk '{print $2,$5}'
  done
eth0: 9001        # old pool underlay
eth0: 9001        # new pool underlay — same, so no heterogeneity here
```

Full-size payloads traverse with DF set, and `ss` already reported `pmtu:1450` equal to the interface MTU with zero retransmissions. **MTU branch closed.** (Note how `ss` pre-answered this: a PMTUD black hole cannot coexist with `retrans:0/0`.)

**Step 4 — rule out the dataplane, one command.**

```console
$ hubble observe --namespace serving --to-namespace feature-store \
    --verdict DROPPED --last 500
(no output)
$ hubble observe --namespace serving --to-namespace feature-store --last 5
Nov 18 09:41:02.114: serving/api-7f9:52488 -> feature-store/fs-6b4d9:8080 to-endpoint FORWARDED (TCP Flags: ACK, PSH)
...
```

Zero drops in the last 500 flows on this path; every flow `FORWARDED`. **Dataplane branch closed** — and note that this also confirms Hubble is actually observing this path, which is the check that distinguishes "no drops" from "Relay is not deployed and I am looking at nothing" (§7).

**Step 5 — the mesh is not on this path**, by design (`feature-store` is unmeshed), so there is no proxy to blame. If it were meshed, this is where `istioctl proxy-config endpoints` and the Envoy response flags would go.

**Step 6 — so: which receivers, and why?** Steps 1–5 say the network is innocent and the receiver's window is closed. Go look at the receivers, and compare the two node pools.

```console
$ kubectl get pods -n feature-store -o wide | awk '{print $7}' | sort | uniq -c
      6 gpu-a3-011      # old pool
      6 gpu-a3-012      # old pool
      2 gpu-c4-101      # NEW pool
$ kubectl exec -n feature-store fs-9d2f7 -- ss -tiem 'sport = :8080' | head -4
ESTAB 131072 0 10.244.9.44:8080 10.244.3.7:52490
     skmem:(r131072,rb131072,t0,tb87040,f0,w0,o0,bl0,d1842)
     cubic rtt:3.402/0.712 rcv_space:14480 rcv_ssthresh:64088
```

**There it is.** On a pod scheduled to the **new** pool: `r131072` equal to `rb131072` — the receive buffer is completely full — and `d1842`, 1,842 packets dropped **by the socket itself**. Cross-check with the kernel's own reason:

```console
$ kubectl debug node/gpu-c4-101 -it --image=busybox -- sh -c \
    'echo 1 > /sys/kernel/tracing/events/skb/kfree_skb/enable;
     timeout 5 cat /sys/kernel/tracing/trace_pipe | grep -c SOCKET_RCVBUFF'
2317
```

`SOCKET_RCVBUFF` — "socket receive buff is full" (§6). 2,317 in five seconds on that node alone.

**The cause:** the new pool has a different CPU generation, and the feature-store process is CPU-bound on deserialisation; on the new nodes it is scheduled with a lower CPU quota inherited from a pool-specific LimitRange, so it drains its socket more slowly, closes its receive window, and every client blocks. **12% of requests** is the fraction that hashed to the two pods on the new pool: 2 of 14 pods ≈ 14%, which matches the observed 12% within noise — and that arithmetic is itself a confirmation, because a network cause would not produce a ratio equal to a pod-count ratio.

**The five-line summary a staff engineer produces:**

```
  Symptom:  p99 40 ms → 2.8 s for ~12 % of serving→feature-store requests.
  NOT:      loss (retrans 0/0 over 3.4 M segs), DNS (no 5 s quantisation,
            200 lookups < 6 ms), MTU (DF-sweep clean, pmtu == link MTU),
            policy/dataplane (0 DROPPED in 500 flows), mesh (not on path).
  IS:       receiver-side application stall. Client rwnd_limited 92 % of
            busy time; server socket rmem == rcvbuf with 1,842 socket drops;
            kernel reason SOCKET_RCVBUFF, 2,317 in 5 s on gpu-c4-101.
  WHY 12 %: 2 of 14 backends live on the new node pool ≈ 14 %.
  ROOT:     pool-specific LimitRange gives feature-store a lower CPU quota
            on the new pool. Fix the LimitRange; add a CPU-quota diff to the
            node-pool onboarding checklist.
  COST OF THE BISECTION: 6 commands, ~4 minutes, no packet capture.
```

**Deliverable:** a one-page network-debugging **decision-tree runbook** — symptom shape → first command → how to read the result → next branch — covering the socket / DNS / MTU / dataplane / LB-mesh / fabric branches, with the fabric branch explicitly annotated *"the tools in this runbook cannot see this traffic; hand off to A02.7's acceptance signals"*, and every branch terminating in either a cause or a named owner.

## Practice

**Task A — build the tree, then test it against history.** Write the runbook from §11 in your own words, one page. Then take the last five network-ish incidents from your team's history and walk each one down the tree on paper. Record, for each, how many steps it would have taken and where the real investigation diverged. The divergences are the branches your tree is missing.

**Task B — the socket-first reflex.** In a test cluster, build a server that accepts connections and deliberately stops calling `read()` after 1 MB. Drive load into it. Capture `ss -tiem` on both ends and identify, from the fields alone, that the receiver is the bottleneck — then repeat with a *sender* whose `SO_SNDBUF` is tiny and show `sndbuf_limited` dominating instead, and with an idle client showing `app_limited`. Three scenarios, three fields, one command.

**Task C — drop reasons.** Enable the `kfree_skb` tracepoint and deliberately create four different drops: a NetworkPolicy denial, an oversized packet with DF set, a full socket receive buffer, and traffic to a port with no listener. Record the `reason` and `location` for each. You now have a lookup table built from your own kernel rather than from a blog post.

**Task D — policy drop, found by identity.** Apply a NetworkPolicy that denies a real path. Find it with `hubble observe --verdict DROPPED`, reading the source/destination **identities** and the denying policy object rather than the IPs. Then delete the client pod, let it come back with a new IP, and show that the same Hubble query still works — and that a query written against the old IP does not.

**Task E — `pwru` with encapsulation.** Trace a packet through the kernel hooks to the exact drop function. Then wrap the path in VXLAN and show that a plain 5-tuple filter loses the packet at the encapsulation boundary, and that `--filter-track-skb` (and/or `--filter-tunnel-pcap-l3`) recovers it. Record the CPU cost of the trace on the node while it runs, so the "when not to use this" judgement is based on your own number.

**Task F — the flagship scenario.** Run [packet-path teardown + debug runbook](../practice/packet-path-and-debug/README.md)'s **"the all-reduce got slower this week."** Inject one of: a NUMA-misaligned reschedule (07), a link in PFC pause, an MTU regression on the RDMA network, or a noisy-neighbour softirq on the storage NIC (A02.1). For each, walk the tree and show which branch it lands on — including the ones where steps 1–5 are all silent, which is the fabric branch's own signature. Narrate each bisection out loud in under a minute per branch; that is the [checkpoint](../checkpoint.md)'s format.

## Common pitfalls

- **Reaching for capture first.** `tcpdump` answers "what was on the wire", and the opening question is almost never that. Steps 0–1 cost nothing and eliminate most of the tree; capture is what you do once you have a specific hypothesis about bytes.
- **Trusting a weak negative.** "I ran `dig` and it worked" does not close the DNS branch for an intermittent problem. Close a branch with a test whose *negative* is structural — a latency distribution with no quantisation, `retrans:0/0` over millions of segments, zero `DROPPED` verdicts over hundreds of flows.
- **Stopping at the first plausible cause.** The Datadog incident is the canonical example: a real contributing factor was found and was not the root cause. Finish the branch you are on and confirm the numbers *explain the whole symptom* — in the worked example, the 12% had to equal a pod-count ratio before the story was closed.
- **Reading a tool's silence without knowing its scope.** `hubble observe` returning nothing may mean no drops, or may mean the agent is node-local and you are asking the wrong node, or that Relay is not deployed. Verify the tool can see the path (query for `FORWARDED` flows) before you treat "no drops" as evidence.
- **Treating a `DROPPED` verdict as a bug.** A default-deny policy denying traffic exactly as designed produces a permanent stream of `Policy denied DROPPED`. The tool reports what and why; whether it should have happened is your judgement, and the useful follow-up is `egress_denied_by`, not an assumption.
- **Believing a host-side capture about MTU or segmentation.** GRO/TSO coalesce below your capture point, so you see 64 KB segments that never existed on the wire (A02.1 §12). Capture on a tap, or disable the offloads for the test — and re-enable them.
- **Running `pwru` broadly in production.** kprobe-based tracing at high packet rates has real CPU cost, and it needs a privileged pod with `/sys/kernel/debug`. Reproduce in staging or scope the filter by netns, interface and function regex.
- **Debugging a meshed path with packet tools.** The client's peer is a sidecar on the same machine. Its RTT, its resets and its socket state describe the proxy hop, not the upstream. Go to the proxy's stats and response flags (06 §7).

## Self-check

- **A pod reports request latencies that are always almost exactly 5 seconds. What does the *quantisation itself* tell you, before you know anything about DNS?** *Answer:* That a **timer** fired. Variable load, a slow backend, congestion and queueing all produce distributions; only a timeout produces repeated identical delays. So the question reduces to "which timer on this path equals 5 s", and on Kubernetes that is glibc's resolver `RES_TIMEOUT` — one of a parallel A/AAAA pair being dropped by a conntrack insertion race, so the resolver waits one full retry interval (mechanism: 02 and 01b.7; fixes: NodeLocal DNSCache, `single-request-reopen`, `ndots:1`, TCP). The transferable move is matching the number to a timer, which works for the 1/3/7 s SYN retry ladder and for client-library timeouts too.

- **Which three `ss -tie` fields partition "why was this connection not faster", and why are they not in the man page?** *Answer:* `rwnd_limited` (blocked by the **peer's** advertised window — the receiving application is slow), `sndbuf_limited` (blocked by our own send buffer — raise `SO_SNDBUF`/`tcp_wmem`), and `app_limited` (we had nothing to send). All three are expressed as time against `busy`, the total time the socket had data to send, so they read as percentages of the connection's active life. They are emitted by `tcp_stats_print()` in iproute2's `misc/ss.c` but are absent from `ss(8)`, which documents only the older field set — a good reason to treat the source as the authority for tool output.

- **What does the `skb:kfree_skb` tracepoint give you that `ethtool -S` and `/proc/net/softnet_stat` cannot, and what is its limitation?** *Answer:* Counters name a *stage* — packets died at the ring, or the backlog, or the qdisc. The tracepoint names the **reason** (a symbolic value from a 200+ entry enum in `include/net/dropreason-core.h`, e.g. `NETFILTER_DROP`, `SOCKET_RCVBUFF`, `PKT_TOO_BIG`, `CPU_BACKLOG`, `TC_INGRESS`) **and the location** (the kernel function and offset, printed with `%pS`), which is a cause rather than a stage. Limitations: it fires for every drop on the box, so it needs a filter on a busy node; it says nothing about packets that were delivered incorrectly; and older or vendor drop sites still report `NOT_SPECIFIED`.

- **A flow log shows `10.0.3.7 → 10.0.9.2` and you are debugging an intermittent cross-service issue that spans a redeploy. Why is this nearly useless, and what replaces it?** *Answer:* Pod IPs live for minutes; by the time a sampled, minutes-delayed flow record surfaces, both addresses may belong to different workloads in different namespaces, so you cannot map the flow to services or reproduce it. Identity-based telemetry replaces it: Cilium/Hubble security identities derived from labels, or mesh SPIFFE IDs, where the flow reads `serving/api → feature-store/fs` and survives churn. Flow logs remain the right artefact for forensics and compliance — long retention, coverage outside the cluster — usually owned by a different team on a different timescale.

- **Why can neither Hubble nor `pwru` diagnose a slow all-reduce, and what do you use instead?** *Answer:* Hubble instruments tc and cgroup eBPF hooks; `pwru` instruments skb-handling kernel functions; `ss` reads socket state. RDMA traffic creates no skb, traverses no tc hook and has no socket, because removing exactly those stages is what RDMA is for (07, 09.3). Their silence on a fabric problem is expected and is itself the tell. Diagnosis moves to the fabric's own signals: `NCCL_DEBUG=INFO`'s `GPU Direct RDMA Enabled/Disabled` lines, `nccl-tests` busbw against the acceptance baseline, and `ethtool -S <pf>` per-priority pause, ECN-mark and discard counters — with discards on a lossless priority required to be exactly zero.

- **You get "the network is broken" and have sixty seconds before the incident call. What do you run, and what will its answer let you say?** *Answer:* `ss -tie` inside an affected client pod, and again on a server pod. If `retrans` is zero over a large `segs_out`, `rtt` is close to `minrtt`, and one of `rwnd_limited`/`sndbuf_limited`/`app_limited` dominates `busy`, you can say — with evidence, not opinion — that the network delivered every byte and the bottleneck is above it, naming which side. That single result reassigns ownership of the incident and eliminates branches 2 through 6 of the tree.

## Connections & what's next

This lesson closes the module's through-line: every hop's mechanism, cost and failure mode from 01 through 07 becomes one branch of a single tree, each branch reachable in one command, and the fabric branch explicitly handed back to 07's acceptance signals and module 09's fabric counters where this lesson's own tooling stops. The socket-level reading in §4 is the practical counterpart of 01's `throughput = cwnd / RTT`; §6's drop reasons sit directly on top of 01b.7's netfilter and conntrack material; §7's identity model is lesson 05's NetworkPolicy identity mechanism seen from the observability side; and §10's proxy boundary is lesson 06's response-flag vocabulary put to work.

It is also the direct rehearsal for the module's [checkpoint](../checkpoint.md), which hands you a symptom and asks for the mechanism, the command and the cost, in this lesson's order. Beyond the checkpoint, the staff move is to make the tree operational rather than documentary: wire each branch's discriminating signal into alerting so the *first* page already carries a branch — "p99 up, retransmissions flat, receive windows closing" rather than "latency is up" — which is the difference between starting a bisection at step 0 and starting it at step 4.

## References & further reading

**Primary sources — read directly from upstream source in this pass**

1. `iproute2` — `misc/ss.c`, `tcp_stats_print()` and `print_skmeminfo()`. Authority for every field name, order and unit in §4, including `busy`, `rwnd_limited`, `sndbuf_limited`, `notsent`, `app_limited`, `delivery_rate` and the `skmem:(r,rb,t,tb,f,w,o,bl,d)` tuple. https://git.kernel.org/pub/scm/network/iproute2/iproute2.git
2. `iproute2` — `man/man8/ss.8`. The documented subset of `-i` fields (units for `rtt`, `rto`, `ato`; `pmtu`, `cwnd`, `ssthresh`, `pacing_rate`, `rcv_space`), which is notably smaller than what the tool prints — the basis for §4's note that the source is the authority. https://git.kernel.org/pub/scm/network/iproute2/iproute2.git
3. Linux kernel — `include/trace/events/skb.h`. The `kfree_skb` tracepoint's arguments (`skb`, `location`, `reason`, `rx_sk`) and its exact `TP_printk` format, reproduced in §6. https://github.com/torvalds/linux/blob/master/include/trace/events/skb.h
4. Linux kernel — `include/net/dropreason-core.h`. The `skb_drop_reason` enum and its per-value documentation; source of §6's table (`NETFILTER_DROP`, `SOCKET_RCVBUFF`, `SOCKET_BACKLOG`, `TCP_LISTEN_OVERFLOW`, `CPU_BACKLOG`, `QDISC_DROP`, `PKT_TOO_BIG`, `IP_OUTNOROUTES`, `NEIGH_*`, `FRAG_*`, `TCP_RFC7323_PAWS`, `NO_SOCKET`, and the note that `QDISC_DROP` has a finer-grained `qdisc:qdisc_drop` tracepoint). https://github.com/torvalds/linux/blob/master/include/net/dropreason-core.h
5. `cilium/cilium` — `api/v1/flow/flow.proto`. The `Verdict` enum, the `DropReason` enum, and the policy-correlation fields `egress_allowed_by` / `ingress_allowed_by` / `egress_denied_by` / `ingress_denied_by` with the `Policy` message's `name`/`namespace`/`labels`/`revision`/`kind`. Source of §7's field list. https://github.com/cilium/cilium/blob/main/api/v1/flow/flow.proto
6. Cilium documentation — `Documentation/observability/hubble/index.rst` and `hubble-cli.rst`, read from the repository. Source of §7's node-local-by-default scope, the Relay requirement for cluster-wide visibility, and the exact `hubble observe --verdict DROPPED` output format. *(`docs.cilium.io` was not fetched; the rendered site and the repository carry the same text.)* https://docs.cilium.io/en/stable/observability/hubble/
7. `cilium/hubble` — `README.md`. Source of the DNS/HTTP/TCP flow-line formats, the `--allowlist`/`--denylist` raw `FlowFilter` mechanism, `--print-raw-filters`, and the `-o json | jq` pattern for filtering on L7 fields. https://github.com/cilium/hubble
8. `cilium/pwru` — `README.md` (the complete flag list, kernel and `CONFIG_*` requirements, and the privileged-pod manifest) and `internal/pwru/output.go` `PrintHeader()` (the exact column layout reproduced in §8). Basis for the §8 correction about `--filter-track-skb` and the tunnel-pcap expressions. https://github.com/cilium/pwru
9. `tcpdump` — `tcpdump.1`. Filter syntax and capture-control flags used in §9's targeted-capture example. https://www.tcpdump.org/manpages/tcpdump.1.html

**Corrections made in this rewrite**

10. The previous version stated that `pwru`'s 5-tuple filter "stops working once a packet is encapsulated … which can force falling back to manually correlating by `skb` pointer identity". `pwru`'s current flag set addresses this directly: `--filter-track-skb` continues tracing a packet after it stops matching the filter (the NAT and encapsulation case), `--filter-track-skb-by-stackid` follows it across `kfree_skb`, and `--filter-tunnel-pcap-l2`/`--filter-tunnel-pcap-l3` match the inner header of VXLAN/Geneve traffic. §8 teaches the flags and keeps skb-pointer correlation as the fallback it now is.
11. The previous version stated that a `DROPPED` verdict "carries the *specific policy identity* that caused the drop, so output can be piped directly to identify the exact `NetworkPolicy` object to edit". The mechanism is narrower and worth stating precisely: the drop reason (`POLICY_DENIED`) and the endpoint identities are always present, while the *policy object* is named by the separate policy-correlation fields (`egress_denied_by` / `ingress_denied_by`), which carry `CiliumNetworkPolicy` metadata on policy-verdict events. §7 describes the fields as they exist in `flow.proto`.

**Real-world engineering blogs**

12. Cloudflare, "Lost in transit: debugging dropped packets from negative header lengths." https://blog.cloudflare.com/lost-in-transit-debugging-dropped-packets-from-negative-header-lengths/ · Mark Betz, "Exhausting conntrack table space crippled our k8s cluster." https://medium.com/@betz.mark/exhausting-conntrack-table-space-crippled-our-k8s-cluster-98564f6f34e0 · Datadog, "It's always DNS . . . except when it's not." https://www.datadoghq.com/blog/engineering/grpc-dns-and-load-balancing-incident/ — *summarised from their published accounts; not re-fetched in this pass, and no number above depends on them.*

**Deeper dives**

13. `microsoft/retina` — cloud- and CNI-agnostic eBPF flow telemetry, usable on EKS/AKS/GKE regardless of dataplane and able to feed Hubble's UI. https://github.com/microsoft/retina · `iovisor/bpftrace` — for one-off aggregations over the tracepoints in §6, e.g. counting drops by reason. https://github.com/iovisor/bpftrace · `retis-org/retis` — a packet-tracing and drop-monitoring tool built on the same tracepoints, with symbol resolution and aggregation. https://github.com/retis-org/retis
