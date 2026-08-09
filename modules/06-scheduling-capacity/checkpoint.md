# 🗓️ Checkpoint — 06 · Scheduling, queueing and capacity

The **completion gate**. Prove it with the [Kueue setup + showback](practice/kueue-showback/)
deliverable and answer the probes cold. You've passed when you can, **unaided**:

## Pass criteria

- [ ] **1 · Deadlock and fix.** Reproduce the 4-replica-on-3-GPU default-scheduler deadlock,
      then show gang scheduling admitting it atomically, and explain the mechanism at the
      scheduler-framework level.
- [ ] **2 · Kueue cold.** Define ClusterQueue, LocalQueue, ResourceFlavor, Cohort, and walk
      borrowing + preemption + fair-sharing from memory — including how Kueue quota differs
      from k8s ResourceQuota.
- [ ] **3 · Effective capacity.** Given a fragmented fleet inventory + job mix, compute usable
      capacity and explain why it's below allocated capacity.
- [ ] **4 · The 128-GPU design.** Lay out quotas for 3 research teams + 1 prod service, and
      **defend** the borrowing and preemption choices on fairness and cost grounds.
- [ ] **5 · The commitment mix.** Propose a reserved/on-demand/spot ladder with a blended
      $/GPU-hr and break-even utilisation, and explain why GPUs can't be autoscaled like CPUs.
- [ ] **6 · The showback report.** Generate a per-ClusterQueue showback table joining queue
      usage to cost, extending `gpu-cost-operator`.
- [ ] **7 · Choose a scheduler.** Given a fleet/tenancy profile, pick Kueue vs Volcano vs KAI
      and justify it.

## Depth probes (answer cold)

- [ ] Why doesn't the default scheduler roll back the partially-placed pods to unblock the cluster?
- [ ] Why does Kueue *suspend* jobs instead of rejecting them like ResourceQuota?
- [ ] `borrowingLimit` vs `lendingLimit` — what does each cap?
- [ ] When does fair-sharing preemption pick a different victim than classic priority preemption?
- [ ] Why does a topology-spread all-reduce gang waste GPU-hours? (tie to 02b interconnect)
- [ ] Why is 90% *allocated* capacity not 90% *usable* capacity?
- [ ] Why is preemption economically useless without checkpointing?
- [ ] At roughly what sustained utilisation does a 1-yr reserved commitment beat on-demand?

## Interview-readiness proxy

- [ ] You have run Kueue with two queues and observed borrowing and preemption yourself.
- [ ] You have produced a per-ClusterQueue showback report.

## Answers / notes

_Record answers as you close each lesson; link the Kueue manifests + showback + 128-GPU doc for items 1–7._
