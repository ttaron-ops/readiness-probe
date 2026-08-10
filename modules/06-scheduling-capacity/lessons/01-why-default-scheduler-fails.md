---
lesson: "06.1"
title: "Why the default scheduler fails distributed jobs"
module: "06"
concept: "default-scheduler-deadlock"
status: not-started
est_time: "6h"
prev: null
next: "02-gang-scheduling.md"
artifacts: []
sources: 7
---

# 06.1 · Why the default scheduler fails distributed jobs

> **Concept.** The default scheduler binds pods one at a time and never reconsiders, so a distributed job that needs all-or-nothing placement can strand GPUs in a partial-placement deadlock — held, idle, and billing.
>
> Module: [🗓️ 06 — Scheduling, queueing and capacity](../README.md) · Deliverable: [Kueue setup + per-queue showback](../practice/kueue-showback/README.md)

## Where this fits
Module 02 made you fluent in the scheduling *cycle* itself: `PreFilter → Filter → PostFilter → PreScore → Score → Reserve → Permit → Bind`, and the extension points a plugin can hook. That module's unit of analysis was always **one pod**. It never had to ask what happens when the unit of *correctness* is a group of pods — four training replicas that only make progress if all four start together. This lesson is module 06's entry point precisely because it closes that gap: it shows, mechanically, why a scheduler built around per-pod decisions produces a structural deadlock for gang-shaped GPU jobs, and it sets up the fix (L2) as the obvious next move rather than a black box you're told to trust.

## Why this matters
"Why can't you just run distributed training on vanilla `kube-scheduler`?" is close to a universal opener in GPU-platform interviews, and the JDs this course targets say so directly — Anthropic's Sr Staff+ Kubernetes Platform posting names "gang scheduling" explicitly, and CoreWeave's Principal/Staff Cluster Orchestration role wants a "technical authority on scheduling, quota enforcement, fairness, pre-emption, and multi-tenant GPU isolation." The answer they're listening for is a mechanism, not an opinion: the scheduler is a **greedy, per-pod, incremental binder with no rollback**. On CPU workloads that greed is invisible; on gang-shaped GPU jobs it produces a deadlock where N-1 replicas pin GPUs while the Nth pends forever.

The economic stakes are concrete and, worse, **silent** — nothing crashes, no alert fires, the bill just keeps running. At list-price H100 rates a single deadlocked 3-GPU hold burns real money every hour with zero training steps produced, and if it strands a full node across two mutually-blocked jobs the number climbs into thousands of dollars a week (worked out below). Because nothing errors, this failure mode is discovered by a human noticing low SM utilization or a job that "seems stuck" — not by a scheduler complaint. That gap between *looks fine on a dashboard* and *is burning money* is exactly why this is the first lesson of a module whose thesis is "every scheduling decision is a cost decision."

## What's new here (calibration)
- **Module 02 (scheduler framework)** — you already know the scheduling cycle and the `Filter`/`Score`/`Reserve`/`Permit` extension points cold. We reference this vocabulary, we don't re-derive it.
- **Module 02b (NVLink/NVSwitch, Topology Manager)** — you already know why co-located replicas want the same interconnect domain. Not repeated here; it resurfaces in L6.
- **Module 04 (GPU quotas, MIG, time-slicing)** — you already know how a GPU becomes a countable `nvidia.com/gpu` resource and how `ResourceQuota` caps it. Also assumed, not re-taught.
- **Genuinely new here:** the systems-level consequence the framework docs never spell out — the scheduling cycle runs **once per pod, in isolation**, with no notion of a pod's peers and no transactional rollback across pods; the **observability trap** where GPU-memory-allocated dashboards can misreport a deadlocked job as healthy; and the fact that **native Kubernetes is closing this gap upstream** via a new `Workload` API (alpha since v1.35, advancing toward beta in v1.36/v1.37) — a forward-pointer that didn't exist when this course was first scoped.

## Core concepts

### The default scheduler is a per-pod greedy binder
`kube-scheduler` pulls one pod off the `activeQ`, runs it through the scheduling cycle, and **binds it**. Then it pulls the next pod. Three properties make this fatal for distributed jobs:

1. **No group awareness.** A pod carrying `app=train, replica=2` is, to the scheduler, an anonymous pod requesting `nvidia.com/gpu: 1`. Nothing in the default cycle asks "are this pod's three siblings also placeable *right now*?" Each replica is judged on its own feasibility, in isolation from its peers.
2. **Greedy commitment.** The moment a pod passes `Filter` and clears `Permit`, the scheduler `Reserve`s the node's resources and binds. That GPU is now consumed. The decision is final for the lifetime of the pod — there is no "tentative" placement that the scheduler later reconsiders in light of what happens to other pods.
3. **No cross-pod rollback.** There is no transaction. Having bound replicas 0–2, the scheduler has no mechanism that says "replica 3 is unschedulable, therefore un-bind 0–2 and free their GPUs." Bound pods stay bound. The scheduler simply leaves replica 3 in `activeQ`/`unschedulableQ` and retries it forever.

The `Reserve` extension point *does* have an `Unreserve` counterpart — but it only fires to undo a reservation for **the same pod** when a *later stage of that same pod's own cycle* fails (e.g. `Permit` timeout, bind error). `Unreserve` has no reach across sibling pods. That asymmetry — rollback exists, but only along the single-pod axis — is the entire bug.

### The partial-placement deadlock
Take a 4-replica distributed job, each replica requesting `nvidia.com/gpu: 1`, on a cluster with exactly **3 free GPUs** spread across nodes. Timeline:

```
t0  replica-0 → Filter passes (node A has 1 GPU) → bind. Free GPUs: 3→2
t1  replica-1 → Filter passes (node B has 1 GPU) → bind. Free GPUs: 2→1
t2  replica-2 → Filter passes (node C has 1 GPU) → bind. Free GPUs: 1→0
t3  replica-3 → Filter: every node fails the `noderesources fit` check
                (0 allocatable nvidia.com/gpu) → Pending, requeued forever.
```

Now replicas 0–2 are `Running` but **doing nothing**. Most collective/distributed frameworks (`torch.distributed`, NCCL rendezvous, MPI) block in `init_process_group` until **all** ranks join. So you have 3 GPUs `Running`, 0 steps computed, and 1 GPU-shaped hole that will never fill *from this job's own actions*.

It gets worse under contention. Suppose a **second** 4-replica job is also waiting. The two jobs can interleave and each grab a partial share (say 2 GPUs each on a 4-GPU-free cluster that needs 4-at-once per job). Neither can ever reach quorum, neither releases, and neither yields — a **mutual deadlock** where the cluster is 100% allocated and 0% productive. This is the canonical starvation pathology gang scheduling exists to prevent, and it is exactly the failure mode OpenAI's infrastructure team names as the reason they built a coscheduling plugin at 7,500-node scale (see Real-world use cases, below).

### Why doesn't it self-heal?
- **Retries don't help.** The scheduler re-queues replica-3 and re-runs its cycle on every cluster event. But the event it's waiting for — a GPU freeing up — can only come from one of *its own siblings exiting*, and they won't exit because the job never starts. Circular.
- **`activeDeadlineSeconds` / job timeouts** eventually fail the Job object, freeing GPUs — but that's a *crash*, minutes-to-hours later, after full billing, and on retry the job walks right back into the same wall.
- **Pod priority/preemption** can evict *lower*-priority pods to make room, but it operates per-pod too: preempting one victim frees one GPU for replica-3, which then binds — and now the *victim's* job is the one deadlocked. Preemption without gang awareness relocates the deadlock; it doesn't dissolve it.

### The observability trap: memory-allocated vs. SM-utilization
This is where the failure hides from you operationally. The three "Running" pods are not idle at the OS level: CUDA contexts are allocated, NCCL has typically already begun its rendezvous handshake (a `TCPStore`-style connection attempt), and framework init (`torch.cuda.init()`, tensor pre-allocation) can show **non-zero GPU memory allocated** even though **SM (streaming multiprocessor) utilization sits at ~0%**. A dashboard keyed off "GPU memory used" or "process count" will happily report this job as *healthy* — it looks exactly like a job doing useful, memory-bound work. Only a dashboard keyed off DCGM's SM-utilization metric (or an equivalent compute-activity signal) distinguishes "deadlocked" from "actually training." This is the direct, practical reason to wire Module 05's GPU observability stack to alert on **sustained near-zero SM utilization with nonzero allocation and Running status** — that specific signature is close to a fingerprint for this failure mode.

### The cost framing: held-but-idle is pure burn
A held GPU bills identically whether it's saturated at 100% SM utilization or blocked in an NCCL barrier. In FinOps terms the deadlock converts **provisioned capacity into stranded capacity** with a utilization of zero and a cost of full list price. On an 8×H100 node at a snapshot rate of **$2.35/GPU-hr** *(2025 neocloud on-demand-tier figure — see L8 for why market segment matters when you quote a $/GPU-hr number)*:

- 3 GPUs held idle in the single-job deadlock = **3 × $2.35 = $7.05/hr** of pure waste, indefinitely.
- If the deadlock strands the *whole* node (two jobs mutually blocked across all 8) = **8 × $2.35 = $18.80/hr ≈ $451/day ≈ $3,158/week** for zero output.

Every scheduling primitive in this module — gang admission, queues, quotas, fair-share, preemption — is ultimately a lever to drive that stranded-capacity number toward zero. That is why "platform engineering meets FinOps" is not a slogan here: **each scheduling decision is a cost decision**, and the default scheduler makes the worst possible one silently.

### Where the fix has always lived (forward pointer to L2)
The scheduler framework already exposes the seams — module 02 gave you the vocabulary, this module puts it to work. A gang plugin intervenes at:
- **`QueueSort`** — keep a group's pods adjacent so they're considered together rather than interleaved with strangers.
- **`PreFilter`** — reject the whole group early if fewer than `minMember` pods could ever fit (the "partial-scheduling guard").
- **`Permit`** — the load-bearing extension point: hold each admitted pod in a *waiting* state and **only approve the batch once `minMember` pods are simultaneously waiting**; otherwise time out and release, so nothing binds partially.
- **`Reserve`/`Unreserve`** — release a group member's reservation if the gang ultimately can't assemble.

L2 builds exactly this. For now, you only need to *see* the deadlock and understand precisely why the framework, as shipped, cannot prevent it on its own.

### New since this course was scoped: native gang scheduling is arriving upstream
For most of Kubernetes' history, "gang scheduling requires a third-party plugin or a full alternate scheduler" was simply true — you either installed `scheduler-plugins`' coscheduling plugin (L2), adopted Kueue/Volcano (L3–L5), or lived with the deadlock. That stopped being unconditionally true in **Kubernetes v1.35** (December 2025), which shipped [KEP-4671 "Gang Scheduling via Workload API"](https://github.com/kubernetes/enhancements/blob/master/keps/sig-scheduling/4671-gang-scheduling/README.md) as **alpha**. The design, confirmed directly from the KEP text:

- A new **`Workload`** object is a static template describing a group's scheduling policy — it doesn't manage pod lifecycles itself.
- A new **`PodGroup`** object is the runtime instance: controllers create `PodGroup`s from `Workload` templates and track live scheduling status against them (this deliberately avoids putting every replica's status on one shared, contended object — and sidesteps etcd's 1.5 MB object-size ceiling on very large gangs).
- The core policy field is `GangSchedulingPolicy.minCount` — read that as the upstream, first-class cousin of the `minMember` you'll configure by hand in L2.
- Pods opt in via a new `spec.schedulingGroup.podGroupName` field.
- The scheduler runs a **Workload Scheduling Cycle** that evaluates all gang members against a single cluster-state snapshot at once, rather than one pod at a time — the atomic "all-or-nothing" semantics baked into the scheduling loop itself instead of bolted on via a plugin's `Permit` hook.
- Per the KEP's own status table: **alpha in v1.35**, with **beta targeted for v1.37**, gated behind a `GenericWorkload` feature flag — meaning it ships **disabled by default**, and Kubernetes v1.36 (May 2026) advanced the implementation further (see kubernetes.io's "Advancing Workload-Aware Scheduling" post) without yet turning it on for everyone.

Two things to hold onto: first, **this is SIG-Scheduling's own admission** that the gap this lesson describes is real and structural, not a workaround for a bug — it's significant enough that core Kubernetes is building a native primitive to close it. Second, **it doesn't make Kueue (L3–L5) redundant**. The native `Workload`/`PodGroup` API solves *atomic admission* — exactly what L2's coscheduling plugin solves today. It says nothing about quota pools, cohorts, borrowing, fair-sharing, or per-team showback — the cost/fairness economics layer this module spends most of its hours on. Read it as: **the gang-admission mechanism is standardizing into core Kubernetes; the queueing and quota-economics layer built on top of it is not, and that's where Kueue's value concentrates.**

## Perspectives

**Developer.** A researcher submitting a multi-worker `Job` sees `torch.distributed.init_process_group` hang with no error — just silence — because the scheduler-level failure (3/4 pods bound) produces no application-level exception; NCCL/Gloo rendezvous simply blocks on the missing rank forever. The developer's mental model ("I asked for GPUs, Kubernetes gives me GPUs, my job runs") breaks the first time this happens, and debugging it *looks* like a networking bug — a rendezvous timeout, a firewall issue — not a scheduling one. Without knowing this lesson's mechanism, the natural instinct is to add more retries and longer timeouts to the training script, which makes the economics strictly worse.

**Operator/platform.** The platform engineer's job is to make this failure *structurally impossible* rather than reactively debugged — the fix is an admission-time invariant (all-or-nothing), not a runbook someone follows at 2am. The operator also has to reason about *interaction* with other pending work: does the greedy binder's per-pod ordering actively starve gangs behind small jobs that "sneak in" to the exact fragments a large gang would have needed? It can, and this tension — one job's atomicity vs. another job's latency — is the thread that runs through the rest of the module.

**Hardware/kernel.** The three "Running" pods are not idle at the OS level — CUDA contexts are allocated, NCCL has usually already begun its rendezvous handshake, and GPU memory shows non-zero allocation from framework init even though SM utilization is ~0%. This is why GPU dashboards that key off *memory allocated* or *process count* can misreport a deadlocked job as "healthy," while SM-utilization/DCGM metrics show the truth — a strong argument for cross-referencing scheduling health signals with Module 05's observability stack rather than trusting either metric alone.

**Economics/FinOps.** This is a "silent" failure mode economically — nothing crashes, nothing alerts, the bill just keeps running. It compounds because retries (`activeDeadlineSeconds` timeouts) don't fix root cause, so a broken job can strand-and-crash-loop, multiplying wasted GPU-hours by however many retry attempts happen before someone notices — which, on a busy platform team, can be a full weekend.

## Real-world use cases

- **OpenAI — "Scaling Kubernetes to 7,500 Nodes"** — https://openai.com/index/scaling-kubernetes-to-7500-nodes/. What it shows: OpenAI built and ran a **gang scheduling plugin** at 7,500-node scale specifically because their biggest jobs run MPI, where any missing participant halts the whole job — alongside "team taints" and CPU/GPU "balloons" for fair distribution. A first-party account of the exact deadlock problem being solved operationally at hyperscale, not theoretically. *(Search-verified this session across multiple independent citations; direct fetch of openai.com was blocked by this session's network egress — cite the canonical URL per SPEC sourcing rules.)*
- **Lambda — "Why your Kubernetes scheduler can't handle AI workloads"** — https://lambda.ai/blog/why-your-kubernetes-scheduler-cant-handle-ai-workloads. What it shows: a GPU-cloud vendor's own engineering explanation that "by default, `kube-scheduler` doesn't gang-schedule — the feature exists in alpha but ships disabled — and it schedules pods one at a time, meaning it has no concept of 'all pods in this job must start together, or none of them start.'" A second, independent restatement of this lesson's thesis in different words — useful as a "read two accounts, same mechanism" exercise. *(Search-verified; fetch blocked by egress this session.)*
- **Kubernetes upstream — "Kubernetes v1.35: Introducing Workload Aware Scheduling"** — https://kubernetes.io/blog/2025/12/29/kubernetes-v1-35-introducing-workload-aware-scheduling/. What it shows: SIG-Scheduling's own confirmation that scheduling a workload "is much more complex than scheduling a single Pod, as it often requires considering all Pods together instead of scheduling each one independently" — the authoritative statement, from the people who own the scheduler, that this lesson's thesis is a structural gap the project itself is now closing. *(Search-verified; fetch blocked by egress this session.)*
- **kubernetes/enhancements — KEP-4671 "Gang Scheduling via Workload API"** — https://github.com/kubernetes/enhancements/blob/master/keps/sig-scheduling/4671-gang-scheduling/README.md. What it shows: the actual design document for native gang scheduling — the `Workload`/`PodGroup` split, `GangSchedulingPolicy.minCount`, and the alpha (v1.35) → beta (targeted v1.37) rollout plan. **Fetched and verified directly this session.**

## Worked example

Reproduce the deadlock on a 3-node kind cluster where each node advertises a fake `nvidia.com/gpu: 1`. Submit a 4-replica job requesting one GPU each:

```bash
$ kubectl apply -f gang4.yaml          # parallelism: 4, each requests nvidia.com/gpu: 1
$ kubectl get pods -l job-name=gang-demo -o wide
NAME              READY   STATUS    NODE                      NOMINATED
gang-demo-abc01   1/1     Running   fake-gpu-worker           <none>
gang-demo-abc02   1/1     Running   fake-gpu-worker2          <none>
gang-demo-abc03   1/1     Running   fake-gpu-worker3          <none>
gang-demo-abc04   0/1     Pending   <none>                    <none>

$ kubectl describe pod gang-demo-abc04 | grep -A3 Events
Events:
  Type     Reason            Message
  ----     ------            -------
  Warning  FailedScheduling  0/4 nodes are available: 3 Insufficient nvidia.com/gpu,
           1 node(s) had untolerated taint {node-role.kubernetes.io/control-plane}.
```

Confirm the burn — three GPUs allocated, one pod pending, no progress:

```bash
$ kubectl get nodes -o json | jq -r '.items[]
    | select(.metadata.name|test("worker"))
    | "\(.metadata.name)  alloc=\(.status.allocatable."nvidia.com/gpu")"'
fake-gpu-worker   alloc=1
fake-gpu-worker2  alloc=1
fake-gpu-worker3  alloc=1
# all consumed by abc01/02/03; abc04 will pend indefinitely.
```

Now put a **retry-storm cost model** on top of that capture. Assume this job carries `activeDeadlineSeconds: 3600` (a 1-hour timeout is a common default for long training jobs) and that nobody notices — a realistic on-call gap over a weekend — so it crash-loops **5 times** before someone investigates, on an 8×H100 node at the same **$2.35/GPU-hr** snapshot rate used above:

```
waste = attempts × timeout_hours × GPUs_stranded × rate
      = 5        × 1             × 3             × $2.35/hr
      = $35.25 stranded, for 5 hours of wall-clock time and zero training steps
```

That's the *narrow* number — just the 3 stranded GPUs, not counting the 3 *legitimately busy-looking* GPUs that were never doing useful work either, or the human time spent diagnosing what looks like a networking bug. Compare it against the same job under native gang admission (either L2's coscheduling plugin or the v1.35+ Workload API): admission is refused in seconds because quorum can't be reached, **zero GPUs are ever bound**, and the failure is visible immediately as "0/4 admitted, waiting for capacity" rather than a silent hang. The contrast is the entire argument for this module: **fast-fail (gang) beats slow-fail (timeout) even before you count the deadlock's own waste, because the timeout mechanism is the *only* self-healing path the default scheduler has, and it's measured in hours, not seconds.**

Capture the `get pods` output and the `describe` Events block: that pair is your L1 artifact and the "before" half of the deliverable's gang-scheduling demo.

## Practice

On a **kind cluster with fake GPUs** (no real hardware):

1. Create a 3-worker kind cluster. Advertise a fake extended resource on each worker by patching node status (kubelet won't invent `nvidia.com/gpu`, so you inject it):
   ```bash
   for n in fake-gpu-worker fake-gpu-worker2 fake-gpu-worker3; do
     kubectl proxy --port=8001 & PROXY=$!; sleep 1
     curl -s --header "Content-Type: application/json-patch+json" -XPATCH \
       "http://localhost:8001/api/v1/nodes/$n/status" \
       --data '[{"op":"add","path":"/status/capacity/nvidia.com~1gpu","value":"1"}]'
     kill $PROXY
   done
   kubectl get nodes -o custom-columns=NAME:.metadata.name,GPU:.status.capacity.'nvidia\.com/gpu'
   ```
   (`~1` is JSON-Pointer escaping for the `/` in `nvidia.com/gpu`.)
2. Submit a `Job` with `spec.parallelism: 4`, `spec.completions: 4`, each pod `resources.limits.nvidia.com/gpu: 1`, running a `sleep 3600` (fake work — we're testing placement, not compute).
3. Watch placement settle: `kubectl get pods -w`. Confirm exactly **3 Running, 1 Pending**.
4. Capture the deadlock: save `kubectl get pods -o wide` and `kubectl describe pod <pending>` (the `FailedScheduling` / `Insufficient nvidia.com/gpu` Events).
5. **Stretch (optional):** if your kind cluster's node image tracks a recent enough Kubernetes minor version, check whether `GenericWorkload` is available as an alpha feature gate (`kube-apiserver --feature-gates=GenericWorkload=true`) and inspect whether a `Workload`/`PodGroup` object is offered by the API — even if you don't wire it up end-to-end, seeing the CRD-free, built-in shape of the object is worth five minutes.

**Acceptance:** a saved capture showing 3 pods bound and holding fake GPUs while the 4th pends with `Insufficient nvidia.com/gpu`, plus a one-line cost annotation (`3 × $2.35/hr = $7.05/hr stranded`). This is the reproduced-deadlock "before" capture for the deliverable's gang demo; L2 produces the "after."

## Common pitfalls

- **Believing `PodDisruptionBudget` or `priorityClass` alone fixes this.** Neither is group-aware; `PodDisruptionBudget` governs *voluntary evictions* of already-running pods, and priority preemption operates per-pod. Neither prevents partial admission in the first place.
- **Assuming a bigger cluster "solves" fragmentation-induced deadlock.** More free GPUs reduce the *odds* of hitting the wall but don't remove the structural gap — the deadlock is about atomicity, not capacity. (This ties directly into L7's fragmentation math: capacity and usable capacity are different numbers.)
- **Confusing "Pending" with "safe."** A pod stuck `Pending` looks inert, but if it already passed `Reserve` on a node before failing a later stage, that node's resources may show as consumed in some tooling even though nothing is bound — don't assume `Pending` always means zero resource impact.
- **Trusting a GPU-memory-allocated dashboard as a proxy for "is this job doing useful work."** As the Core concepts section shows, a deadlocked job can show nonzero memory allocation from framework init while SM utilization sits at zero — cross-check with a compute-activity metric, not an allocation metric.

## Self-check

- Why doesn't the default scheduler roll back the 3 already-bound pods to unblock the cluster? **Answer:** Because the scheduling cycle is per-pod and greedy with no cross-pod transaction. Binding is a final commitment; the only rollback hook, `Unreserve`, undoes a reservation for the *same* pod when a later stage of *its own* cycle fails — it has no visibility into or authority over sibling pods. The scheduler has no concept that replicas 0–2 belong to the same job as the pending replica 3, so there is nothing to trigger or scope a group rollback.
- What does this deadlock cost per hour on an 8×H100 node at $2.35/GPU-hr, and what does 5 retries at a 1-hour timeout add up to? **Answer:** The 3 held-but-idle GPUs burn 3 × $2.35 = **$7.05/hr** for zero output; a full-node mutual deadlock is 8 × $2.35 = **$18.80/hr** (~$451/day, ~$3,158/week). Five retries against a 1-hour `activeDeadlineSeconds` timeout adds up to 5 × 1hr × 3 GPUs × $2.35/hr = **$35.25** of stranded compute for a single job over a weekend on-call gap, on top of the deadlock's own waste before the first timeout fires.
- Why does Kubernetes ship native gang scheduling as alpha-and-disabled in v1.35/v1.36 rather than turning it on by default, and what does that imply? **Answer:** A new admission-time gate (`GenericWorkload`) that changes how groups of pods get scheduled is a behavior change with real blast-radius risk for existing per-pod-scheduled workloads at cluster scale — SIG-Scheduling ships it opt-in, gated, and iterating toward beta (targeted v1.37) precisely so operators can adopt it deliberately rather than have scheduling semantics shift under them on a routine upgrade.
- Why would a GPU-memory-allocated dashboard metric under-report this failure while an SM-utilization (DCGM) metric would catch it? **Answer:** A deadlocked pod's process has already initialized CUDA and often begun NCCL rendezvous, which allocates GPU memory even though no kernel is executing — so memory-allocated stays nonzero and looks "in use," while SM utilization, which reflects actual compute activity, sits near 0%. The mismatch between "memory used" and "SM busy" is the fingerprint of this specific failure mode.
- Name two scheduler-framework extension points (from Module 02) where a gang plugin could intervene, and say which one does the real work. **Answer:** **`Permit`** is the load-bearing one — it holds each pod in a waiting state and approves the batch only once `minMember` pods are simultaneously waiting, the core all-or-nothing gate. **`PreFilter`** supports it by rejecting the whole group early if fewer than `minMember` could ever fit, saving wasted `Reserve` cycles. `QueueSort` (keep group pods adjacent) and `Reserve`/`Unreserve` (release a member's reservation if the gang can't assemble) also participate.

## Connections & what's next

This lesson is the diagnosis; L2 is the cure. The `Permit`-phase mechanism sketched in "Where the fix has always lived" is exactly what L2 wires up end-to-end, replica by replica, on the same deadlocked cluster you just built. The "bigger cluster doesn't fix fragmentation" pitfall above resurfaces with real numbers in L7's fragmentation/effective-capacity math, and the observability trap (memory-allocated vs. SM-utilization) is worth keeping in your back pocket all the way to L8, where checkpoint-survivable preemption depends on correctly reading GPU activity signals. **Next: [02 — Gang scheduling: all-or-nothing admission](02-gang-scheduling.md)**, which takes this exact cluster and this exact 4-replica job and shows the `PodGroup`/`minMember` mechanism turning "3 running, 1 stranded" into "4 pending together, 0 GPUs wasted, then 4 running together."

## References & further reading

**Primary sources**
- [scheduler-plugins — coscheduling README](https://github.com/kubernetes-sigs/scheduler-plugins/blob/master/pkg/coscheduling/README.md) — the canonical "why gang" motivation and the `PodGroup`/`Permit` design; read for the maintainers' own statement of the deadlock problem and the fix L2 installs. **Fetched and verified this session.**
- [kubernetes/enhancements — KEP-4671, Gang Scheduling via Workload API](https://github.com/kubernetes/enhancements/blob/master/keps/sig-scheduling/4671-gang-scheduling/README.md) — read for the native `Workload`/`PodGroup` design, `GangSchedulingPolicy.minCount`, and the alpha→beta rollout plan. **Fetched and verified this session.**
- [Kubernetes v1.35 — "Introducing Workload Aware Scheduling"](https://kubernetes.io/blog/2025/12/29/kubernetes-v1-35-introducing-workload-aware-scheduling/) — read for SIG-Scheduling's own framing of why per-pod scheduling is insufficient for workloads. *(Search-verified; fetch blocked by egress this session — cite per SPEC sourcing rules.)*
- [Kubernetes v1.36 — "Advancing Workload-Aware Scheduling"](https://kubernetes.io/blog/2026/05/13/kubernetes-v1-36-advancing-workload-aware-scheduling/) — read for what changed between v1.35 alpha and the v1.36 iteration on the way to beta. *(Search-verified; fetch blocked by egress this session.)*

**Real-world engineering blogs**
- [OpenAI — "Scaling Kubernetes to 7,500 Nodes"](https://openai.com/index/scaling-kubernetes-to-7500-nodes/) — what it shows: a gang scheduling plugin built and run in production at 7,500-node scale, specifically to stop MPI jobs deadlocking on partial placement. *(Search-verified; fetch blocked by egress this session.)*
- [Lambda — "Why your Kubernetes scheduler can't handle AI workloads"](https://lambda.ai/blog/why-your-kubernetes-scheduler-cant-handle-ai-workloads) — what it shows: a GPU-cloud vendor's independent restatement of the same deadlock mechanism, plus its own comparison of Kueue vs. Volcano as fixes. *(Search-verified; fetch blocked by egress this session.)*

**Deeper dives**
- [CNCF — "Kubernetes as AI's operating system: 1.35 release signals"](https://www.cncf.io/blog/2026/02/23/kubernetes-as-ais-operating-system-1-35-release-signals/) — a broader industry read on why workload-aware scheduling is a signal about where core Kubernetes is heading for AI infrastructure, beyond this one KEP. *(Search-verified this session.)*
- Module 02 (this course) — the scheduling cycle and `Filter`/`Score`/`Reserve`/`Permit` extension points; reread its cycle diagram alongside the deadlock timeline above and the hole becomes obvious.
