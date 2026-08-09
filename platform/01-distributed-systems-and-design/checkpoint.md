# 🧩 Checkpoint — Distributed systems & system design

The **completion gate**. Prove it with the [staff design portfolio](practice/staff-design-portfolio/)
and answer cold. The bar is not "I can explain it" — it's **"I can quantify the tradeoff and
defend the number."** For any system named, in **< 5 minutes**, unaided:

## Pass criteria

- [ ] **1 · Class it in PACELC.** State the consistency/availability class and where reads *and*
      writes actually land (e.g. etcd = PC/EC, quorum reads via ReadIndex; K8s watch cache =
      eventual unless forced).
- [ ] **2 · Do the back-of-envelope.** Estimate the dominant resource (QPS / GPU-seconds / HBM-GB /
      bandwidth) from first principles and identify the **first bottleneck** — the estimate must
      *choose* the architecture.
- [ ] **3 · Name a failure + its blast-radius control.** One gray/correlated failure mode and the
      containment (cells / shuffle-sharding / checkpoint interval) that bounds it.
- [ ] **4 · Consistency, precisely.** Linearizable vs serializable as orthogonal axes; give a
      system that is one but not the other.
- [ ] **5 · etcd under load.** Why it's fsync-bound, the quorum math (3 vs 5 vs 7), and how a slow
      disk becomes fleet-wide scheduling latency.
- [ ] **6 · Overload.** State Little's Law and why you *shed* rather than *defer*; name a
      metastable-failure trigger.
- [ ] **7 · Exactly-once, honestly.** Why true exactly-once needs 2PC or an idempotent sink; design
      the idempotency key for a GPU billing pipeline.
- [ ] **8 · Drive a design.** Run the 8-step method on a GPU-platform prompt, volunteering
      scale/cost/failure/SLO tradeoffs by name.

## Depth probes (answer cold)

- [ ] What can a K8s `resourceVersion=0` read cause, and what can it *not*? (double-schedule vs lost write)
- [ ] Raft ReadIndex vs a log append — how do you serve a linearizable read cheaply?
- [ ] 3-member vs 5-member etcd — failures tolerated and the added quorum-RTT cost.
- [ ] Why is `hash mod N` partitioning a trap, and what do production stores do instead?
- [ ] Cache stampede on a hot-key expiry — the single-flight + jittered-TTL fix.
- [ ] Young/Daly: the optimal checkpoint interval for a 1,024-GPU job, and why.
- [ ] Which "plane" is a given prompt in (control / training / serving), and how does that change the answer?

## Interview-readiness proxy

- [ ] You have 5–6 staff design write-ups, each with a guarantees table, a BOTE, and a blast-radius section.
- [ ] Given any system, you class it in PACELC and find its first bottleneck in minutes.
- [ ] You reach for the right tradeoff axis *by name* before the interviewer asks.

## Fail signal

- [ ] "I can explain it" without a number · draws boxes without estimating · can't name the failure
      mode or its blast radius · treats consistency as one axis instead of two.

## Answers / notes

_Record answers as you close each lesson; link the staff design portfolio for items 1–8._
