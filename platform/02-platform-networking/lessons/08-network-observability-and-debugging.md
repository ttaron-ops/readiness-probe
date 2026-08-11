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
sources: 7
---

# A02.8 · Network observability and debugging

> **Concept.** Staff network debugging is disciplined bisection down a decision tree, not tool-flailing — and at fleet scale the tree runs on identity-based eBPF flow telemetry, not IP-based packet sampling.
>
> Module: [🌐 Platform networking depth](../README.md) · Track A — Platform excellence

## Where this fits

This lesson closes the module's through-line. Every prior lesson put a mechanism, a cost, and a failure mode on one hop — the kernel datapath (01), DNS (02), the LB (03), the cloud boundary (04), the K8s dataplane (05), the mesh (06), and the deliberately-bypassed RDMA path (07). None of that is useful in an incident unless you can bisect *across* all of them fast, under pressure, without knowing in advance which hop is guilty. That's the actual staff skill this module is building toward, and it's also the checkpoint's format: given a symptom, name the exact mechanism and the exact command. This lesson is where every earlier lesson's failure signature becomes one branch of a single decision tree, including the one branch — the RDMA fabric — where the tools taught here (Hubble, pwru) structurally cannot reach, because RDMA bypasses the kernel network stack they instrument.

## Why this matters

The senior-to-staff jump in network debugging is not more tools — it's a *procedure* that converges. Given "intermittent timeouts in prod," a staff engineer bisects: DNS? MTU? conntrack? LB/mesh? fabric? — each with a one-command test and a characteristic signature — instead of tcpdumping randomly. This is also the highest-signal interview payoff: the tree *is* the answer. And at fleet scale the old dataplane (sample-and-grep IPs) is broken because pod IPs churn faster than you can query them; the staff move is identity-based flow telemetry from the eBPF layer, where a flow is "service A → service B," not "10.0.3.7 → 10.0.9.2 (already gone)."

## What's new here (calibration)

- We do not re-teach tcpdump/wireshark capture and filters, ping/traceroute/mtr, reading a raw cloud flow log, that eBPF can observe the kernel, or basic "is the port open" triage — assumed baseline.
- New here: the decision tree treated explicitly as a *transferable staff discipline* (differential diagnosis, not a networking-specific trick), with the cost/overhead tradeoffs of the tools that implement each branch — when eBPF tracing is the right first move and when it isn't, and when identity-based telemetry is the right tool versus when IP-based flow logs still are.
- New here: the fabric branch of the tree is named as a hard boundary, not a gap to paper over — Hubble and pwru instrument the kernel network stack, and RDMA's entire value proposition (07, module 09) is bypassing that stack. Diagnosing the fabric branch means falling back to 07's acceptance-test signals (busbw, PFC counters, `NCCL_DEBUG=INFO`), not reaching for a kernel-tracing tool that literally cannot see the traffic.

## Core concepts

**The debugging decision tree IS the deliverable.** Symptom shapes the first branch:

- **Is it DNS?** Signature: latency in exact **5-second multiples** → conntrack race on parallel UDP `A`/`AAAA` lookups from the same socket (the classic musl/Alpine + kube-dns DNAT insert race; UDP replies collide, one is dropped, the resolver waits the 5s timeout). `NXDOMAIN`-then-success or many failed lookups → **ndots:5** in the pod's `resolv.conf` walking the search list (`svc.cluster.local`, `cluster.local`, …) before the real name. Tests: `dig` the FQDN with a trailing dot vs bare; check `resolv.conf` ndots; watch for the 5s quantization.
- **Is it MTU?** Signature: small requests work, large payloads **hang** (not reset) — a PMTU black hole. An overlay/tunnel (VXLAN/IPsec/WireGuard) shrinks the usable MTU (~50 bytes for VXLAN) but a firewall drops the ICMP "fragmentation needed," so the sender never learns. Handshake and small GETs succeed; the first full-window transfer stalls. Test: ping with DF bit at descending sizes to find the real MTU; compare overlay MTU vs fabric MTU.
- **Is it conntrack?** Signature: random drops under load, `nf_conntrack: table full, dropping packet` in `dmesg`. Either the table is exhausted (`nf_conntrack_count` near `_max`) or DNAT/insert races under high new-flow churn. Test: read count/max, watch the dmesg line, check `insert_failed`/`drop` in `conntrack -S`.
- **Is it the LB/mesh?** Signature: retries, sudden latency cliffs, or partial availability tied to one service. Envoy/mesh mechanics: outlier detection **ejecting** a healthy-looking host, **panic mode** (too many hosts unhealthy → the LB ignores health and sends to all), sidecar startup races, retry amplification. Test: mesh/proxy stats (ejections, panic, 5xx by upstream), not the app logs.
- **Is it the fabric?** Signature: RDMA/collective slowness, no app-level errors, and — the tell that you're on this branch — Hubble and pwru show nothing wrong, because RDMA traffic never crosses the kernel hooks they instrument. Fall back to PFC pause counters, ECN marks, and `NCCL_DEBUG=INFO` collective logs (callback to A02.7's acceptance signals: busbw, channel count).

Teaching this tree — symptom → first command → likely cause — is the interview move and the runbook's spine. The order matters as much as the branches: cheapest, most-discriminating test first, DNS and MTU before conntrack, conntrack before mesh, mesh before fabric — each test should rule a branch *in or out* with one command, not require standing up new tooling.

**eBPF-based observability replaces sampling — for the branches it can see.** Cloud flow logs are sampled, delayed minutes, and keyed on IPs that no longer exist; AWS VPC Flow Logs in particular apply a variable, often-undocumented per-flow sampling rate for very high-packet-rate flows, so even absent an explicit sampling config a flow log is not a complete packet-level record. The eBPF dataplane sees every packet at the kernel hook with **identity**:

- **Cilium Hubble** — L3-L7 flow visibility from the eBPF dataplane, no sidecar. `hubble observe --verdict DROPPED` shows not just that a packet dropped but *why* — e.g. `Policy denied` carrying the *specific policy identity* that caused the drop, not just a generic "denied" — so `hubble observe --verdict DROPPED` output can be piped directly to identify the exact `NetworkPolicy` object to edit, collapsing what used to be a multi-step search into one command. Service maps, DNS- and HTTP-level visibility included.
- **Microsoft Retina** — cloud- and CNI-agnostic counterpart; works without Cilium and can feed Hubble's UI, so you get the same flow telemetry on EKS/AKS/GKE regardless of dataplane.
- Contrast: cloud flow logs are sampled, delayed, and IP-not-identity — the right tool for forensics and compliance (and usually owned/queried by a security or cloud team on that timescale), not for live on-call bisection. Hubble is the live bisection tool; it typically has its own retention/export limits shorter than compliance-grade flow logs, so the two are complementary — Hubble does not replace the need for flow logs, it replaces reaching for them *first*.

**Packet capture at scale is a different discipline.** You cannot tcpdump a fleet. Targeted, eBPF-triggered capture instead:

- **`pwru`** (packet, where are you?) — traces a single packet's path through *every* kernel network hook (netfilter, tc, routing, XDP), so you see exactly which hook dropped it. The tool for "the packet leaves but never arrives." It filters by kernel function name and packet metadata (5-tuple) and attaches to a large fraction of `kfree_skb`/networking kprobes — but it has real, practical limits: filters that match on the original 5-tuple stop working once a packet is encapsulated (the tuple you're matching on no longer exists in the encapsulated frame), which can force falling back to manually correlating by `skb` pointer identity across hook boundaries. It also has a real cost: kprobe-based tracing at high packet rates carries non-trivial CPU overhead, which is why standard practice is to reproduce the failure in staging or scope the filter as narrowly as possible rather than run it broadly across live production traffic.
- **BPF drop monitors** — `dropwatch`, `retis` — attach to `kfree_skb`/drop tracepoints and report drop location + reason kernel-wide without a full capture; cheaper first move than a full `pwru` trace when you just need "where are drops happening," not "trace this one packet."
- Sampled/rolling captures for the cases you must see on the wire.
- **Offloads distort host captures (A02.1 callback):** GRO/TSO mean a host-side tcpdump shows coalesced 64KB "segments" that never existed on the wire; the NIC segments/reassembles below your capture point. Capture on the wire (SPAN/tap) or account for it when MTU/segmentation is the question.

**Latency/loss diagnosis — separate the three causes.**
- **Loss** vs **queueing (bufferbloat)** vs **app stall**. `ss -ti` on the socket gives `retrans`, `rtt`/`rttvar`, and `cwnd` — rising retrans = loss; high RTT with no retrans and a large cwnd = queueing/bufferbloat; healthy socket stats with slow progress = the app isn't reading/writing (stall).
- RTT histograms via `bpftrace`; per-hop attribution to localize.
- For RDMA/lossless, "the collective is slow" enters through PFC pause counters, ECN marks, and `NCCL_DEBUG=INFO` — same fabric branch as A02.7, and the same tools-that-can't-reach-it boundary noted above.

**Identity vs IP is the fleet-scale reframe.** In Kubernetes an IP is valid for the life of a pod — minutes. By the time a flow log surfaces `10.0.3.7 → 10.0.9.2`, both pods may be gone and reassigned. Staff-grade telemetry keys on stable identity: Cilium/Hubble security identities (label-derived), or mesh **SPIFFE** IDs. A flow reads "`checkout` → `payments`," survives churn, and is the only thing that lets you debug an intermittent issue spanning redeploys.

## Perspectives

**Procedural/cognitive.** The decision tree is a differential-diagnosis discipline, borrowed structurally from clinical reasoning: each branch has a cheap, low-false-positive test, and the branches are ordered cheapest-and-most-likely-to-rule-out first. This is a generalizable staff-engineering skill that outlasts any specific tool in the tree — the same discipline applies to debugging a build pipeline or a data-corruption bug, and interviewers who ask "how would you debug X" are usually testing for this ordering instinct, not for tool trivia.

**Tooling-lifecycle.** eBPF tools are themselves running code with their own overhead, not a free window into ground truth. `pwru`'s kprobe attachment has real, non-trivial cost at very high packet rates — knowing when *not* to reach for full-fidelity kernel tracing, and when a sampled or targeted capture is the right first move, is as much a staff skill as knowing the tool exists at all.

**Cross-team-handoff.** Identity-based (Hubble/SPIFFE) vs IP-based (VPC flow logs) telemetry isn't only a technical distinction — it's an organizational one. Flow logs are usually owned and queried by cloud/security teams on a compliance or forensics timescale (hours to months later, audit-grade); Hubble is the live on-call bisection tool with shorter retention. Knowing which team owns which artifact, and which timescale each is built for, is what keeps an incident bisection from stalling on "let me go file a ticket for the flow log query."

**GPU-fleet.** The fabric branch of the tree is the one branch where this lesson's tooling genuinely doesn't reach — Hubble and pwru instrument the kernel network stack, and RDMA's entire point (07) is bypassing that stack via GPUDirect DMA. Diagnosing "the collective is slow" means cross-referencing 07's acceptance-test signals (busbw, `NCCL_DEBUG=INFO` channel counts) and module 09's fabric counters (PFC pause, ECN marks) — an explicit, named boundary of this lesson's own toolkit, not a gap to quietly paper over.

## Real-world use cases

- **Cloudflare, "Lost in transit: debugging dropped packets from negative header lengths."** A real production incident where Cloudflare used `pwru` in staging to trace an IPVS-native-encapsulation bug to the exact kernel drop point, leading to an upstream Linux kernel patch — a near-perfect worked-example match for this lesson's `pwru` deliverable, including hitting the exact encapsulation-filter limitation named above. https://blog.cloudflare.com/lost-in-transit-debugging-dropped-packets-from-negative-header-lengths/
- **Mark Betz, "Exhausting conntrack table space crippled our k8s cluster."** A bisection narrative — symptom, then `dmesg` "table full," then root cause, then fix — that mirrors this lesson's "is it conntrack?" branch move for move. https://medium.com/@betz.mark/exhausting-conntrack-table-space-crippled-our-k8s-cluster-98564f6f34e0
- **Datadog, "It's always DNS . . . except when it's not."** A staff engineer bisecting the DNS branch, then the conntrack branch, then the LB/client-retry branch of essentially this lesson's own decision tree, converging on the real, non-obvious root cause instead of stopping at the first plausible one. https://www.datadoghq.com/blog/engineering/grpc-dns-and-load-balancing-incident/

## Worked example

Build the runbook by breaking things, in order:
1. **Policy drop:** apply a NetworkPolicy that denies a real path; find it with `hubble observe --verdict DROPPED` — read the source/dest **identities** and the `Policy denied` reason (which names the specific policy), not the IPs.
2. **MTU regression:** set the overlay MTU wrong (or block ICMP frag-needed); reproduce the large-payload hang while small requests succeed; diagnose via DF-bit ping sweep, not the app.
3. **conntrack full:** drive new-flow churn until `nf_conntrack: table full` appears in dmesg; catch the drops in `conntrack -S`.
4. **`pwru`:** trace one of the dropped packets through the kernel hooks to name the exact drop point — and, deliberately, wrap it in an encapsulation (VXLAN) to reproduce the Cloudflare case's filter limitation and confirm you know the manual-correlation fallback.

**Deliverable:** a one-page network-debugging **decision-tree runbook** — symptom → first command → likely cause — covering the DNS / MTU / conntrack / LB-mesh / fabric branches, with the fabric branch explicitly annotated as "tools in this runbook cannot see this traffic; fall back to A02.7's acceptance signals."

## Practice

Produce the decision-tree runbook and the four break-and-find exercises above. Then run the flagship scenario in [packet-path teardown + debug runbook](../practice/packet-path-and-debug/README.md): **"the all-reduce got slower this week."** Force the full tree — a NUMA-misaligned reschedule (A02.7), a single link in PFC pause, an MTU regression on the RDMA net, or a noisy-neighbor softirq on the storage NIC (A02.1) — and show which branch each maps to, using Hubble/flow telemetry + fabric counters + an nccl-tests busbw regression as the combined evidence stack. This is also the closest rehearsal you'll get for the module checkpoint's live oral-bisection format: practice narrating symptom → first command → likely cause out loud, in under a minute per branch.

## Common pitfalls

- **"More logging/more tools converges faster."** A fixed decision tree with cheap, ordered first tests converges faster than tool-flailing — the Datadog incident shows even a sophisticated team chasing the wrong hypothesis before tree-discipline (not more dashboards) found the real cause.
- **"eBPF flow data replaces the need for sampled flow logs entirely."** Hubble typically has its own shorter retention/export window, not kept as long as compliance-grade flow logs. They're complementary — Hubble for live bisection, flow logs for forensics/compliance — not a replacement.
- **"A drop verdict from Hubble/pwru always means a bug."** Many drops are intentional and correct — a NetworkPolicy denying traffic exactly as designed. The tool tells you *what* dropped and *why*, not whether the drop itself is wrong; that judgment is still yours.
- **"You can tcpdump your way to ground truth on any host."** GRO/TSO-coalesced captures aren't wire-accurate — a host-side tcpdump shows segments that never existed on the wire. Combine with a SPAN/tap or explicitly disable the offload for the test.
- **"pwru has negligible overhead so it's safe to run broadly in production."** kprobe-based tracing at high packet rates has real CPU cost. Standard practice, per the Cloudflare case, is to reproduce in staging or scope the filter narrowly — not run it fleet-wide against live traffic.

## Self-check

- A pod sees intermittent request latencies that are always almost exactly 5 seconds. What's your first hypothesis and why 5s specifically? **Answer:** A DNS conntrack race on parallel UDP A/AAAA lookups (classic musl + kube-dns DNAT insert race). The 5s is the resolver's UDP retry timeout — one of the two racing replies is dropped by a conntrack insert collision, so the client waits one full timeout before retrying. The quantization to a fixed multiple is the tell that it's a timeout, not variable load.
- Why does a host-side tcpdump mislead you when MTU/segmentation is the question, and what do you do instead? **Answer:** GRO/TSO offloads coalesce packets below the capture point, so tcpdump shows 64KB "segments" that never existed on the wire — you can't see real fragmentation or the true on-wire MTU. Capture on the wire (SPAN/tap), disable the offload for the test, or use a drop monitor / `pwru` that reads the actual kernel path.
- At fleet scale, why is a cloud flow log showing `10.0.3.7 → 10.0.9.2` nearly useless for debugging an intermittent cross-service issue, and what replaces it? **Answer:** Pod IPs live minutes; by the time the sampled, delayed flow log surfaces, both IPs are likely reassigned, so you can't map the flow to services or reproduce it. Replace it with identity-based eBPF telemetry — Hubble security identities or mesh SPIFFE IDs — where the flow is "serviceA → serviceB" and survives pod churn and redeploys.
- Why can't Hubble or pwru diagnose a slow all-reduce on the RDMA fabric, and what do you use instead? **Answer:** RDMA's entire value proposition is bypassing the kernel network stack via GPUDirect DMA — Hubble and pwru both instrument kernel network hooks, so RDMA traffic never crosses the points they observe. Diagnosis falls back to the fabric's own signals: PFC pause counters, ECN marks, and `NCCL_DEBUG=INFO` collective/channel output, cross-referenced with the busbw acceptance baseline from A02.7.
- What's the practical limitation of `pwru`'s 5-tuple filtering that the Cloudflare incident ran into, and how did they work around it? **Answer:** A 5-tuple filter matches the original packet's header, but once a packet is encapsulated (IPVS native encapsulation, in Cloudflare's case) that original 5-tuple no longer exists in the frame the later kernel hooks see, so the filter stops matching post-encapsulation. The workaround was manually correlating the packet across hooks by `skb` pointer identity instead of relying on the 5-tuple filter throughout the whole trace.

## Connections & what's next

This lesson closes the module's through-line: every hop's mechanism, cost, and failure mode from 01 through 07 becomes one branch of the single decision tree taught here, and the fabric branch explicitly hands back to 07's acceptance signals and module 09's fabric counters where this lesson's own tooling can't reach. It is also the direct rehearsal for the module's [checkpoint](../checkpoint.md) — a live oral bisection where you're handed a symptom and have to name the exact mechanism, the exact command, and the cost, in the same order this lesson teaches. Beyond the checkpoint, the next move for a staff engineer is turning this runbook into an actual on-call artifact: wire the decision tree's branches into your team's alerting so the *first* page already carries the likely branch, not just "latency is up."

## References & further reading

**Primary sources**
- Cilium Hubble documentation. https://docs.cilium.io/en/stable/observability/hubble/
- `cilium/pwru` — packet, where are you? https://github.com/cilium/pwru

**Real-world engineering blogs**
- Cloudflare, "Lost in transit: debugging dropped packets from negative header lengths." https://blog.cloudflare.com/lost-in-transit-debugging-dropped-packets-from-negative-header-lengths/
- Mark Betz, "Exhausting conntrack table space crippled our k8s cluster." https://medium.com/@betz.mark/exhausting-conntrack-table-space-crippled-our-k8s-cluster-98564f6f34e0
- Datadog, "It's always DNS . . . except when it's not." https://www.datadoghq.com/blog/engineering/grpc-dns-and-load-balancing-incident/

**Deeper dives**
- `microsoft/retina` — cloud- and CNI-agnostic eBPF flow telemetry. https://github.com/microsoft/retina
- `iovisor/bpftrace`. https://github.com/iovisor/bpftrace
