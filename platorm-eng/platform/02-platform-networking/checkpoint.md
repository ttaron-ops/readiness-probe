# 🌐 Checkpoint — Platform networking depth

The **completion gate**. Prove it with the [packet-path teardown + debug runbook](practice/packet-path-and-debug/)
and a live oral bisection. You've passed when you can, **unaided**:

## Pass criteria

- [ ] **1 · Trace one packet end to end** — app `write()` → socket buffer → qdisc → NIC → CNI
      overlay → Service VIP DNAT → LB → backend, naming the mechanism at each hop and where the
      RDMA path *bypasses* it.
- [ ] **2 · Name what actually limits throughput** — state `throughput = cwnd / RTT`, the BDP,
      and when softirq/RSS steering (not bandwidth) is the ceiling; contrast CUBIC vs BBR.
- [ ] **3 · Bisect an intermittent latency incident live** — "5s API latency on a GPU cluster" →
      walk DNS (ndots / the conntrack race) → MTU (PMTU black hole) → conntrack table → LB/mesh →
      fabric, naming the *first command* at each branch.
- [ ] **4 · Resolve a Service VIP three ways** — iptables kube-proxy vs IPVS vs eBPF/Cilium
      (socket-level DNAT before a packet exists), and say what conntrack does in each.
- [ ] **5 · Put a price + latency on every hop** — same-AZ vs cross-AZ (billed each direction) vs
      cross-region vs egress; explain why egress is the hidden AI bandwidth bill.
- [ ] **6 · Operate RDMA on K8s** — what the Network Operator pieces do, the DRA-vs-device-plugin
      migration, and why a NUMA-misaligned NIC silently halves NCCL bandwidth.
- [ ] **7 · Judge a service mesh** — what it buys vs costs, sidecar vs ambient, and why it must
      never sit on the RDMA data path.

## Depth probes (answer cold)

- [ ] Why does `tcpdump` show 64KB packets that never hit the wire?
- [ ] What causes the ~5s DNS lookup on a busy node, and three fixes?
- [ ] What does Maglev give you that ring consistent-hashing and modulo hashing don't?
- [ ] Overlay MTU: 1500 − VXLAN header = ? and what symptom does a mismatch produce?
- [ ] Why can't the DRA SR-IOV driver and the SR-IOV device plugin run at once?
- [ ] How many NCCL channels does a SYS-distant NIC get vs a NUMA-aligned one, and the bandwidth hit?
- [ ] `hubble observe --verdict DROPPED` — what does it tell you that a VPC flow log can't?

## Interview-readiness proxy

- [ ] You have an annotated end-to-end packet-path teardown (with the RDMA bypass drawn).
- [ ] You have a symptom→first-command→cause network-debugging runbook.
- [ ] Given a cluster network symptom, you bisect it live to the real kernel/CNI/fabric mechanism.

## Fail signal

- [ ] Blames "the network" without bisecting · can't say what limits throughput beyond "bandwidth"
      · treats RDMA as ordinary CNI traffic · can't name a single conntrack/MTU/ndots failure mode.

## Answers / notes

_Record answers as you close each lesson; link the packet-path teardown + debug runbook for items 1–7._
