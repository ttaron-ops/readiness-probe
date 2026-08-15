# 🖥️ Checkpoint — 10 · Bare metal and cluster lifecycle

The **completion gate**. Prove it with the [Capex-vs-cloud model + KTHW/etcd writeup](practice/capex-vs-cloud/)
and answer the probes cold. You've passed when you can, **unaided**:

## Pass criteria

- [ ] **1 · Provision from scratch.** Produce a working control plane via Kubernetes-the-Hard-Way
      and correctly name each cert and which component presents it to which — no notes.
- [ ] **2 · Recover etcd (timed).** Given a quota-exceeded read-only cluster, restore write
      capability (`defrag` + `alarm disarm`); given a destroyed data dir, restore from a snapshot
      with the etcd-3.6 tooling (`etcdutl snapshot restore`). Control plane writable again **in < 30 min**.
- [ ] **3 · Design HA.** Diagram a 3-node stacked-etcd control plane with a kube-vip VIP and state
      the correct upgrade order + version-skew constraints.
- [ ] **4 · Close the health loop.** Describe (or demo) NPD → node condition → cordon/drain → RMA,
      and justify a time-to-detect target against the goodput cost.
- [ ] **5 · Defend the economics.** Present the 64-GPU crossover model and answer "at what
      utilisation does owning beat renting, and what kills the payback?" with **your own numbers**.
- [ ] **6 · Serve traffic without a cloud.** Explain MetalLB BGP vs L2 failover and when
      keepalived/L2 is insufficient (multi-rack).

## Depth probes (answer cold)

- [ ] Which cert does the apiserver present to etcd, and which to the kubelet?
- [ ] 3-member vs 5-member etcd — how many failures does each tolerate, and why?
- [ ] Compaction vs defragmentation — what does each reclaim?
- [ ] Why does etcd hate slow disks — what latency dominates, and what metric warns first?
- [ ] In what order do you upgrade apiserver / controller-manager / kubelet?
- [ ] What does a bare-metal CAPI provider (Metal3/Ironic) do that a cloud provider doesn't?
- [ ] How do you drain a node running a 500-GPU training job without losing it? (tie to 06/08)
- [ ] Roughly what aggregate GB/s keeps 64 H100s fed, and why does metadata-server design matter?

## Interview-readiness proxy

- [ ] You have completed Kubernetes-the-Hard-Way end to end.
- [ ] You have broken etcd deliberately and restored it.
- [ ] You have a written capex-versus-cloud model with your own numbers.

## Fail signal

- [ ] Can consume a cluster but can't rebuild the control plane or recover etcd unaided.

## Answers / notes

_Record answers as you close each lesson; link the KTHW/etcd writeup + capex model for items 1–6._
