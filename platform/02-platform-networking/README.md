---
id: "A-02"
track: "A — Platform excellence"
title: "Platform networking depth"
notion: null                # repo-native module (added in the 12–15mo rebuild), not from the original Notion plan
phase: "Track A · deepen module"
effort: "6–8 weeks ≈ ~32 hrs @ 10–12 hrs/wk"
status: not-started        # not-started | in-progress | checkpoint-passed
prerequisites: []
unlocks: []
started: null
completed: null
---

# 🌐 Platform networking depth

> **Goal.** Deepen networking from the cloud VPC through Kubernetes to the GPU/RDMA fabric, so
> you can **design and debug it end to end** at a staff bar.
>
> **Track A — Platform excellence.** A deepen module: you cleared the senior bar here already;
> this takes it to staff depth and interview-ready fluency.

## Through-line

**Follow one packet, and put a mechanism, a cost, and a failure mode on every hop it takes.** The
module walks a request from the wire and up the kernel datapath (L1), through name resolution
(L2), across load balancers (L3) and cloud boundaries (L4), into the K8s dataplane (L5) and
optional mesh (L6) — then contrasts that ordinary path with the *deliberately bypassed* RDMA path
GPUs use (L7), and closes with how you **observe and debug** all of it (L8). Every lesson answers
the same three staff questions: *what actually limits it, how it fails at fleet scale, and what it
costs* (in latency, dollars, or blast radius).

## Calibrated to your background — what we skip

You can already draw the TCP stack, subnet in your head, and configure a CNI, so we **skip the
fundamentals** and spend only on the **mechanism underneath the abstraction** and the failure mode
that only shows at scale. Lesson 07 **references** your module-09 GPU-fabric artifact (RoCE/IB,
PFC/ECN/DCQCN, GPUDirect, NCCL, rail-optimized Clos) and stays on the **platform-integration and
operations** plane — provisioning and operating RDMA on K8s, not re-deriving fabric physics.

## Lessons

| # | Lesson | Staff delta |
|---|--------|-------------|
| 01 | [TCP/IP and the packet path](lessons/01-tcpip-and-the-packet-path.md) | the kernel datapath; what *actually* limits throughput; BBR; XDP |
| 02 | [DNS and service discovery](lessons/02-dns-and-service-discovery.md) | DNS as a failure amplifier; ndots tax; the 5s conntrack race |
| 03 | [Load balancing](lessons/03-load-balancing.md) | the LB as a failure domain; Maglev; DSR; slow-start/panic mode |
| 04 | [Cloud networking](lessons/04-cloud-networking.md) | a price + latency on every hop; egress as the hidden AI bill |
| 05 | [Kubernetes networking](lessons/05-kubernetes-networking.md) | how a Service VIP resolves (iptables/IPVS/eBPF); conntrack; MTU |
| 06 | [Service mesh](lessons/06-service-mesh.md) | what a mesh buys vs costs; sidecar vs ambient; when NOT to |
| 07 | [GPU and RDMA networking](lessons/07-gpu-and-rdma-networking.md) | operating RDMA on K8s; Network Operator/DRA; NUMA alignment |
| 08 | [Network observability and debugging](lessons/08-network-observability-and-debugging.md) | the symptom→cause decision tree; eBPF (Hubble/pwru) |

Total ≈ **32 hrs ≈ 6–8 weeks**. **Spine:** L1 (datapath), L5 (K8s dataplane), L7 (RDMA ops),
L8 (the debug tree).

## Deliverable & checkpoint

- Build the **[packet-path teardown + debug runbook](practice/packet-path-and-debug/)** — two
  reinforcing artifacts: (1) a single **annotated packet-path teardown** tracing one request end
  to end (app `write()` → socket → qdisc → NIC → CNI overlay → Service VIP DNAT → LB → backend,
  with the RDMA path drawn as the bypass), each hop labeled with mechanism, p99 latency
  contribution, and $; and (2) a **network-debugging decision-tree runbook** mapping symptom →
  first command (`ss -ti`, `hubble observe --verdict DROPPED`, `conntrack -L`, `pwru`,
  `nccl-tests`, PFC counters) → likely cause, explicitly covering the GPU-cluster failure modes.
- The [**checkpoint**](checkpoint.md) is the gate — a live oral bisection: given "intermittent 5s
  API latency" or "the all-reduce dropped 40% overnight," name the exact mechanism and the exact
  command, and quantify the cost.

## Directory layout

| Path | What goes here |
|------|----------------|
| [`lessons/`](lessons/) | One page per concept — notes, worked example, practice, self-check. |
| [`practice/`](practice/) | Design write-ups, labs, diagrams — the buildable output. |
| [`resources/`](resources/) | Saved references, papers, link index. |
| [`checkpoint.md`](checkpoint.md) | Checkpoint answers (the completion gate). |
