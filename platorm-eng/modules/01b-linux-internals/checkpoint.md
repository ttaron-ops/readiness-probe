# 🐧 Checkpoint — 01b · Linux systems internals

The **completion gate**. Prove each with the [Anatomy of a Container](practice/anatomy-of-a-container/)
deliverable and answer the probes cold. You've passed when you can, **unaided**:

## Pass criteria

- [ ] **1 · Locate any container's ns + cgroup state by hand.** Given a PID, produce
      its namespaces (`lsns` / `/proc/<pid>/ns`) and its full cgroup resource picture
      (`cpu.max`, `cpu.weight`, `memory.max`, `memory.high`) with **no** Docker/kubectl.
- [ ] **2 · Trigger and fully explain an OOM kill from the kernel log alone.** Victim,
      cause, cgroup- vs global-scoped, and the role of `oom_score_adj` / working-set.
- [ ] **3 · Diagnose a saturated node via PSI, not utilization.** Show a node `<100%`
      util but resource-pressured; name the pressured resource and justify from `*.pressure`.
- [ ] **4 · Explain a throttled-but-idle pod.** Map a K8s CPU limit to `cpu.max` and read
      `cpu.stat` throttling counters (`nr_throttled`, `throttled_usec`).
- [ ] **5 · Reproduce and reason about conntrack exhaustion**, and state why eBPF
      datapaths avoid per-packet conntrack.
- [ ] **6 · Produce a flame graph and run ≥3 bpftrace one-liners** to answer "what is this
      process actually doing," and articulate the USE method for an unknown-slow node.

## Depth probes (answer cold)

- [ ] Why can load average exceed core count while CPU is ~0%? What is a D-state process?
- [ ] What exactly is "a container" in kernel terms? (namespaces + cgroup + rootfs)
- [ ] A pod has `limits.cpu: 500m` — what value lands in `cpu.max`, and how does CFS bandwidth control throttle it at low average CPU?
- [ ] `memory.high` vs `memory.max` — which does a K8s memory *limit* set, and what's the behavioral difference?
- [ ] "some" vs "full" PSI pressure — precisely?
- [ ] Why does K8s evict on working-set rather than RSS?
- [ ] When is THP `always` a throughput win vs a latency hazard?
- [ ] Why does cross-NUMA memory access hurt a GPU data path?
- [ ] What produces "nf_conntrack: table full, dropping packet"?
- [ ] Why is eBPF safe to run in the kernel? (the verifier)
- [ ] Walk the USE method for a node reported "slow."

## Interview-readiness proxy

- [ ] Whiteboard "what happens, kernel-mechanism by mechanism, when a pod hits its memory limit."
- [ ] Explain "why is this GPU idle while its data-loader pod is throttled."

## Answers / notes

_Record answers as you close each lesson; link the deliverable evidence for items 1–6._
