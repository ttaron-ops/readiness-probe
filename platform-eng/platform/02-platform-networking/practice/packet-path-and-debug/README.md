# Packet-path teardown + network-debugging runbook — Networking module deliverable

Two reinforcing artifacts that together prove staff-level networking fluency: one shows you
understand the *mechanism* of every hop, the other shows you can *bisect a failure* to it. Both
double as interview/portfolio pieces. Finishable on a small cluster (kind/k3s + Cilium) — no GPU
fabric required for most of it.

## A) The annotated packet-path teardown

A single diagram + document tracing **one request end to end**, each hop labeled with (i) the
mechanism, (ii) the p99 latency contribution, (iii) the $ where applicable:

```
app write() → socket send buffer → qdisc (fq/fq_codel) → NIC (TSO/GSO) → wire
   → CNI overlay (VXLAN/Geneve, MTU 1500−50) → node → Service VIP DNAT (iptables/IPVS/eBPF)
   → conntrack entry → LB (Maglev/DSR) → backend pod → app read()
```

- Draw the **RDMA path as the bypass**: GPU → NIC (GPUDirect) → leaf → spine, skipping the kernel
  TCP stack, the overlay, conntrack, and the mesh entirely — and say *why* each is bypassed.
- Label the cost of the hops that cost money (cross-AZ egress, NAT-GW processing) and the ones
  that cost latency (double-proxy mesh hop, overlay encap).

## B) The network-debugging decision-tree runbook

The symptom→cause bisection from lesson 08, with the **first command per branch**:

| Symptom | First check | Command | Likely cause |
|---------|-------------|---------|--------------|
| intermittent 5s latency | DNS | `dig +short`, histogram | ndots storm / conntrack race |
| works small, hangs large | MTU | `ping -M do -s 1472` | overlay/tunnel PMTU black hole |
| random drops under load | conntrack | `conntrack -S`, `dmesg` | table full / insert race |
| resets on scale event | LB/mesh | Envoy logs, `istioctl` | draining / outlier ejection / panic |
| all-reduce slowed | fabric | `nccl-tests`, PFC counters | PFC pause / NUMA-misaligned NIC / flapping link |

Build it by **breaking things**: inject a NetworkPolicy drop and find it with
`hubble observe --verdict DROPPED`; misset an overlay MTU and diagnose the large-payload hang;
fill conntrack and catch the `table full`; use `pwru` to trace one dropped packet through the
kernel hooks.

## Suggested layout

```
packet-path-and-debug/
├── packet-path.md       # the end-to-end teardown (with the RDMA bypass drawn)
├── packet-path.png      # the annotated diagram
├── debug-runbook.md     # the symptom→first-command→cause decision tree
└── README.md            # how it fits + how each break was reproduced
```

## Acceptance criteria (matches the [checkpoint](../../checkpoint.md))

- [ ] a one-request packet-path teardown with the mechanism, p99, and $ per hop
- [ ] the **RDMA path drawn as the bypass**, with why each ordinary hop is skipped
- [ ] a symptom→first-command→cause **decision-tree runbook** covering DNS/MTU/conntrack/LB-mesh/fabric
- [ ] at least three failures **actually reproduced** (NetworkPolicy drop via Hubble, MTU hang, conntrack full)
- [ ] the GPU-cluster failure modes (NUMA-misaligned NIC, PFC pause, storage-NIC softirq) named

## Guardrails

- Most of it runs on a laptop cluster (kind/k3s + Cilium); the RDMA/NCCL branch can be documented
  from module 09 + a 2-GPU `nccl-tests` run if hardware is available, otherwise reasoned.
- **Publishable-by-default** — the packet-path teardown and the debug runbook are strong blog/
  portfolio pieces; scrub any employer-specific topology before posting.
