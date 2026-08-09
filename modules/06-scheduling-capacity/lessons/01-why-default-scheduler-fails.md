---
lesson: "06.1"
title: "Why the default scheduler fails distributed jobs"
module: "06"
concept: "Why the default scheduler fails distributed jobs"
status: not-started
est_time: "4h"
artifacts: []
---

# 06.1 · Why the default scheduler fails distributed jobs

> **Concept.** The default scheduler binds pods one at a time and never reconsiders, so a distributed job that needs all-or-nothing placement can strand GPUs in a partial-placement deadlock — held, idle, and billing.
>
> Module: [🗓️ 06 — Scheduling, queueing and capacity](../README.md) · Deliverable: [Kueue setup + showback](../practice/kueue-showback/README.md)

## Why this matters
"Why can't you just run distributed training on vanilla `kube-scheduler`?" is the opening question of every GPU-platform interview, and the answer is a mechanism, not an opinion: the scheduler is a **greedy, per-pod, incremental binder with no rollback**. On CPU workloads that greed is harmless; on gang-shaped GPU jobs it produces a deadlock where N-1 replicas pin GPUs while the Nth pends forever. Those pinned GPUs are the most expensive idle resource in your fleet — at H100 rates a single deadlocked node quietly burns four figures a week while producing exactly zero training steps. This lesson makes you reproduce that failure so the rest of the module (gang scheduling, queueing, fair-share, preemption) reads as the obvious fix rather than ceremony.

## What's new here
Module 02 taught the **scheduler framework itself**: the scheduling cycle, the `Filter`/`Score` plugin phases, and the `Reserve`/`Permit` extension points where a plugin can hold or reject a pod before binding. Module 02b covered **NVLink/NVSwitch topology** (why co-located replicas want the same domain), and Module 04 covered **`ResourceQuota`, MIG, and time-slicing** (how a GPU becomes a countable `nvidia.com/gpu` resource). You already own all of that vocabulary.

What this lesson adds is the **systems-level consequence** the framework docs never spell out: the scheduling cycle runs **once per pod, in isolation**, with **no notion of a pod's peers** and **no transactional rollback across pods**. The framework gives you the hooks to fix this; the default plugin set does not use them for gang semantics. So the *layer* this module builds — PodGroups, queues, cohorts, quota-aware admission — exists precisely to add the atomic, group-aware, cost-aware admission that the base scheduler structurally cannot provide. We are not re-teaching the cycle; we are showing you the hole in it.

## Core notes

### The default scheduler is a per-pod greedy binder
`kube-scheduler` pulls one pod off the `activeQ`, runs it through the scheduling cycle (PreFilter → Filter → PostFilter → PreScore → Score → Reserve → Permit → binding cycle), and **binds it**. Then it pulls the next pod. Three properties make this fatal for distributed jobs:

1. **No group awareness.** A pod carrying `app=train, replica=2` is, to the scheduler, an anonymous pod requesting `nvidia.com/gpu: 1`. Nothing in the default cycle asks "are this pod's three siblings also placeable *right now*?" Each replica is judged on its own feasibility.
2. **Greedy commitment.** The moment a pod passes `Filter` and clears `Permit`, the scheduler `Reserve`s the node's resources and binds. That GPU is now consumed. The decision is final for the lifetime of the pod.
3. **No cross-pod rollback.** There is no transaction. Having bound replicas 0–2, the scheduler has no mechanism that says "replica 3 is unschedulable, therefore un-bind 0–2 and free their GPUs." Bound pods stay bound. The scheduler simply leaves replica 3 in `activeQ`/`unschedulableQ` and retries it forever.

The relevant `Reserve` extension point *does* have an `Unreserve` counterpart — but it only fires to undo a reservation for **the same pod** when a *later stage of that same pod's cycle* fails (e.g. `Permit` timeout, bind error). `Unreserve` has no reach across sibling pods. That asymmetry is the entire bug.

### The partial-placement deadlock
Take a 4-replica distributed job, each replica requesting `nvidia.com/gpu: 1`, on a cluster with exactly **3 free GPUs** spread across nodes. Timeline:

```
t0  replica-0 → Filter passes (node A has 1 GPU) → bind. Free GPUs: 3→2
t1  replica-1 → Filter passes (node B has 1 GPU) → bind. Free GPUs: 2→1
t2  replica-2 → Filter passes (node C has 1 GPU) → bind. Free GPUs: 1→0
t3  replica-3 → Filter: every node fails the `noderesources fit` check
                (0 allocatable nvidia.com/gpu) → Pending, requeued forever.
```

Now replicas 0–2 are `Running` but **doing nothing** — most collective/distributed frameworks (`torch.distributed`, NCCL rendezvous, MPI) block in `init_process_group` until **all** ranks join. So you have 3 GPUs `Running`, 0 steps computed, and 1 GPU-shaped hole that will never fill *from this job's own actions*.

It gets worse under contention. Suppose a **second** 4-replica job is also waiting. The two jobs can interleave and each grab a partial share (say 2 GPUs each on a 4-GPU-free cluster that needs 4-at-once per job). Neither can ever reach quorum, neither releases, and neither yields — a **mutual deadlock** where the cluster is 100% allocated and 0% productive. This is the canonical starvation pathology gang scheduling exists to prevent.

### Why doesn't it self-heal?
- **Retries don't help.** The scheduler re-queues replica-3 and re-runs its cycle on every cluster event. But the event it's waiting for — a GPU freeing up — can only come from one of *its own siblings exiting*, and they won't exit because the job never starts. Circular.
- **`activeDeadlineSeconds` / job timeouts** eventually fail the Job object, freeing GPUs — but that's a *crash*, minutes-to-hours later, after full billing, and it just re-runs into the same wall on retry.
- **Pod priority/preemption** can evict *lower*-priority pods to make room, but it operates per-pod too: preempting one victim frees one GPU for replica-3, which then binds — and now the *victim's* job is the one deadlocked. Preemption without gang awareness relocates the deadlock; it doesn't dissolve it.

### The cost framing: held-but-idle is pure burn
A held GPU bills identically whether it's saturated at 100% SM utilization or blocked in an NCCL barrier. In FinOps terms the deadlock converts **provisioned capacity into stranded capacity** with a utilization of zero and a cost of full list price. On an 8×H100 node at **$2.35/GPU-hr**:

- 3 GPUs held idle in the single-job deadlock = **3 × $2.35 = $7.05/hr** of pure waste, indefinitely.
- If the deadlock strands the *whole* node (two jobs mutually blocked across all 8) = **8 × $2.35 = $18.80/hr ≈ $451/day ≈ $3,158/week** for zero output.

Every scheduling primitive in this module — gang admission, queues, quotas, fair-share, preemption — is ultimately a lever to drive that stranded-capacity number toward zero. That is why "platform engineering meets FinOps" is not a slogan here: **each scheduling decision is a cost decision**, and the default scheduler makes the worst possible one silently.

### Where the fix will live (forward pointer)
The framework already exposes the seams. A gang plugin intervenes at:
- **`QueueSort`** — keep a group's pods adjacent so they're considered together.
- **`PreFilter`** — reject the whole group early if fewer than `minMember` pods could ever fit ("pre-filter guard").
- **`Permit`** — the load-bearing one: hold each admitted pod in a *waiting* state and **only approve the batch once `minMember` pods are simultaneously waiting**; otherwise time out and release, so nothing binds partially.
- **`Reserve`/`Unreserve`** — release a group member's reservation if the gang ultimately can't assemble.

L2 builds exactly this. For now, you only need to *see* the deadlock.

## Worked example

Reproducing on a 3-node kind cluster where each node advertises a fake `nvidia.com/gpu: 1`. Submit a 4-replica job requesting one GPU each:

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

The three `Running` pods never exit on their own, `abc04` re-queues on every scheduler cycle and always fails the fit check, and the cluster sits at 100% GPU allocation / 0% useful work — the deadlock, captured. Capture the `get pods` output and the `describe` Events block: that pair is your L1 artifact and the "before" half of the deliverable's gang demo.

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

**Acceptance:** a saved capture showing 3 pods bound and holding fake GPUs while the 4th pends with `Insufficient nvidia.com/gpu`, plus a one-line cost annotation (`3 × $2.35/hr = $7.05/hr stranded`). This is the reproduced-deadlock "before" capture for the deliverable's gang demo; L2 produces the "after."

## Self-check
**(a) Why doesn't the default scheduler roll back the 3 already-bound pods to unblock the cluster?**
**Answer:** Because the scheduling cycle is per-pod and greedy with no cross-pod transaction. Binding is a final commitment; the only rollback hook, `Unreserve`, undoes a reservation for the *same* pod when a later stage of *its own* cycle fails — it has no visibility into or authority over sibling pods. The scheduler literally has no concept that replicas 0–2 belong to the same job as the pending replica 3, so there is nothing to trigger or scope a group rollback.

**(b) What does this deadlock cost per hour on an 8×H100 node at $2.35/GPU-hr?**
**Answer:** The 3 held-but-idle GPUs burn 3 × $2.35 = **$7.05/hr** for zero output. If two gang jobs mutually deadlock and strand the full node, it's 8 × $2.35 = **$18.80/hr** (~$451/day, ~$3,158/week) at 0% useful utilization — a held GPU bills at full price regardless of SM activity.

**(c) Name two scheduler-framework extension points (from 02) where a gang plugin could intervene.**
**Answer:** **`Permit`** (hold each pod in a waiting state and approve the batch only once `minMember` pods are simultaneously waiting — the core all-or-nothing gate) and **`PreFilter`** (reject the whole group early if fewer than `minMember` could ever fit). `QueueSort` (keep group pods adjacent) and `Reserve`/`Unreserve` (release a member's reservation if the gang can't assemble) are also valid.

## Resources
1. **scheduler-plugins — coscheduling README** — https://github.com/kubernetes-sigs/scheduler-plugins/blob/master/pkg/coscheduling/README.md — the canonical "why gang" motivation and the PodGroup/Permit design; **deep** read. It states the partial-scheduling/deadlock problem in the maintainers' own words and previews the exact fix L2 installs — the single best primary source for this lesson's thesis.
2. **Module 02 — the scheduler framework** (this course) — the scheduling cycle and `Filter`/`Score`/`Reserve`/`Permit` extension points; **skim** to refresh. This lesson is the "so what" that motivates those hooks — reread 02's cycle diagram alongside the deadlock timeline above and the hole becomes obvious.
