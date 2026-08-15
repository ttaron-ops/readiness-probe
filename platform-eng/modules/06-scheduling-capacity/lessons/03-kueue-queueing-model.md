---
lesson: "06.3"
title: "Kueue I — the queueing model: suspend, admit, and the quota pool"
module: "06"
concept: "Kueue's queueing model: suspend, admit, and the quota pool"
status: not-started
est_time: "10h"
prev: "02-gang-scheduling.md"
next: "04-kueue-cohorts-borrowing-preemption.md"
artifacts: []
sources: 12
---
# 06.3 · Kueue I — the queueing model: suspend, admit, and the quota pool

> **Concept.** Kueue admits work by *suspending* Jobs and *unsuspending* them when quota is free — a queue, not a wall.
>
> Module: [🗓️ 06 — Scheduling, queueing and capacity](../README.md) · Deliverable: [Kueue setup + per-queue showback](../practice/kueue-showback/README.md)

## Where this fits

06.1 showed *why* the default scheduler deadlocks a distributed job that can't get all its pods bound at once. 06.2 fixed the atomicity problem for a single job with gang/co-scheduling: all pods or none. But gang scheduling only answers "can this one job start safely" — it says nothing about *which* job should start next when ten teams are all asking for GPUs that don't exist yet. That's a queueing and quota-accounting problem, and it's the gap this lesson fills. Kueue is the layer that sits above admission and decides, cluster-wide, who gets to run now and who waits — and it's the mechanism that turns "gang-admit this job" into "gang-admit this job *when its team's fair share of the fleet allows it*." Master this lesson and 06.4 (cohorts, borrowing, preemption) and you have covered the deepest interview surface in the whole module.

## Why this matters

You run a fixed GPU fleet that many teams want more of than exists. The stock Kubernetes answer to "team X wants more than its share" is `ResourceQuota`: the API server rejects the over-quota pod **synchronously, at create time**, with a `403 exceeded quota`. The workload is simply gone — nothing remembers it wanted to run, there's no ordering, no fairness, no "run it when capacity frees." For a shared, expensive, permanently-oversubscribed GPU fleet that is the wrong primitive: you want a **queue** that holds work and admits it in order as GPUs free up, so the fleet stays hot and teams get fairness instead of a `kubectl apply` retry storm.

That queue is Kueue, and it is *the* batch-admission layer for GPU Kubernetes in production, not a lab curiosity. CoreWeave — named explicitly in this module's target JDs — runs Kueue on CoreWeave Kubernetes Service (CKS) for "world-class AI labs" doing training and batch-type inference. Netflix rolled Kueue out fully into production managing **millions of batch workloads**, replacing a homegrown scheduler. IBM Research's Vela and Blue Vela GPU supercomputers use Kueue's queueing and cohort model explicitly because, in their own framing, "the challenge is not about getting more GPUs, but getting more out of the GPUs they already have" — which is precisely the FinOps thesis of this module. An interviewer at any GPU-fleet operator can reasonably expect you to define ClusterQueue, LocalQueue, ResourceFlavor, and Workload cold, and to explain *why* Kueue queues instead of rejects. Fumbling that is a strong negative signal for a role whose job description literally says "quota enforcement, fairness, pre-emption."

## What's new here (calibration)

- **You already know controllers, informers, and CRDs (module 02)** — Kueue is not a new paradigm, it's a well-factored Go controller applying the pattern you've already studied to one new domain object (`Workload`) and one clever native-Kubernetes lever (`Job.spec.suspend`). We don't re-teach reconcile loops here.
- **You already know GPU quotas via `ResourceQuota` (module 04)** — we don't re-teach what `ResourceQuota` is; we spend the lesson on *why it's the wrong tool* for a shared, oversubscribed GPU fleet and what Kueue does differently.
- **New here:** the four Kueue objects and how they compose; the suspend/admit state machine as the actual mechanism (not just "Kueue schedules things"); ResourceFlavor design as a real operational decision; and — genuinely new since this lesson was last written — **`AdmissionFairSharing`**, promoted to beta-and-default in Kueue v0.15, a *within-ClusterQueue* fairness mechanism that is easy to confuse with the *cohort-level* `fairSharing` you'll learn in 06.4. They are different knobs solving different problems, and interviewers who've read the recent release notes will ask you to distinguish them.

## Core concepts

### The mechanism: `suspend`

Kubernetes `Job` has a native field `spec.suspend` (bool, GA since 1.24). When `true`, the Job controller creates **no pods** and terminates any that exist; the Job sits inert. Kueue weaponizes this field instead of inventing a new admission path:

1. You submit a Job carrying the label `kueue.x-k8s.io/queue-name: <local-queue>`.
2. Kueue's mutating webhook **forces `spec.suspend: true`** on admission of the Job object (even if you set it false). No pods are created. No GPUs are consumed.
3. Kueue creates a `Workload` object representing the Job's total resource ask (pods × per-pod requests, summed).
4. The Kueue scheduler evaluates the `Workload` against its LocalQueue → ClusterQueue quota. If quota is available in some ResourceFlavor, it **reserves quota** and writes an admission onto the `Workload`.
5. On admission, Kueue **sets `spec.suspend: false`** and injects the flavor's `nodeSelector`/tolerations. *Now* the Job controller creates pods, which the kube-scheduler binds to matching nodes.
6. On completion, quota is released and the next queued `Workload` is evaluated.

That's the whole model. Rejection never happens for capacity reasons — the `Workload` waits with condition `QuotaReserved=False` and an event like `couldn't assign flavors`. This is the direct answer to "why does Kueue suspend jobs instead of rejecting them like `ResourceQuota`": **suspend preserves intent and ordering so the fleet can be oversubscribed safely**, instead of forcing teams into a submit-and-pray retry loop.

### The four objects

| Object | Scope | Role | Who owns it |
|---|---|---|---|
| **ResourceFlavor** | cluster | Maps abstract quota to a *node shape* via labels/taints (e.g. `gpu-type=a100`) | platform |
| **ClusterQueue** | cluster | The **quota pool** — how much of each resource exists, per flavor | platform |
| **LocalQueue** | namespace | Team-facing handle that points at one ClusterQueue | platform, used by teams |
| **Workload** | namespace | Auto-created per Job; the thing that gets queued/admitted | Kueue |

**ResourceFlavor** — a named class of capacity. Quota is always denominated *per flavor*, because "8 GPUs" is meaningless if some are A100 and some are H100. The flavor carries `nodeLabels` that Kueue injects as a `nodeSelector` when it unsuspends the Job, steering pods onto the right nodes.

```yaml
apiVersion: kueue.x-k8s.io/v1beta1
kind: ResourceFlavor
metadata:
  name: a100
spec:
  nodeLabels:
    gpu-type: a100          # must match a label on your GPU nodes
  # nodeTaints / tolerations optional: toleration injected on admission
```

**ClusterQueue** — the pool. `resourceGroups` list the covered resources and, per flavor, the `nominalQuota` (the guaranteed floor this queue owns). `namespaceSelector: {}` means "any namespace's LocalQueue may target me."

```yaml
apiVersion: kueue.x-k8s.io/v1beta1
kind: ClusterQueue
metadata:
  name: team-research
spec:
  namespaceSelector: {}          # {} = all namespaces allowed
  resourceGroups:
  - coveredResources: ["nvidia.com/gpu"]
    flavors:
    - name: a100
      resources:
      - name: "nvidia.com/gpu"
        nominalQuota: 8          # this pool owns 8 A100s
```

`nominalQuota` is the **guaranteed floor** — the number of GPUs this ClusterQueue can always admit against, independent of any other queue. (In 06.4 you add cohorts, and a queue may *borrow* above its nominal from siblings; without a cohort, nominal is a hard ceiling too.)

**LocalQueue** — the namespaced pointer teams actually submit to. Teams never touch ClusterQueues; they submit to a LocalQueue in their own namespace, which routes to a ClusterQueue.

```yaml
apiVersion: kueue.x-k8s.io/v1beta1
kind: LocalQueue
metadata:
  namespace: research
  name: research-lq
spec:
  clusterQueue: team-research
```

This indirection is the RBAC/multi-tenancy seam: teams get `create` on Jobs and see their LocalQueue; only platform edits ClusterQueues and Flavors. It is also the **showback seam** — usage rolls up per ClusterQueue, which is why the deliverable's showback table is keyed on ClusterQueue, not namespace.

### The admission flow, in `kubectl`

```
$ kubectl create -f job-8gpu.yaml
$ kubectl get workloads -n research
NAME               QUEUE         RESERVED   ADMITTED   AGE
big-training-abcd  research-lq   True       True       5s     # quota was free
huge-sweep-efgh    research-lq   False      False      3s     # waiting: pool full

$ kubectl describe workload huge-sweep-efgh -n research
...
Events:
  Warning  Pending  couldn't assign flavors to pod set main:
           insufficient quota for nvidia.com/gpu in flavor a100, request > maximum capacity
```

When `big-training` finishes, `huge-sweep` flips `RESERVED=True`, `ADMITTED=True`, its Job unsuspends, pods appear. You watched a queue drain — the thing `ResourceQuota` cannot do.

### Contrast, sharpened

`ResourceQuota` and Kueue are **complementary, not redundant**, but Kueue is the one you queue with:

- `ResourceQuota` fires at **kube-apiserver admission**, per namespace, on *committed* pod usage. It is a safety ceiling — good for capping total namespace footprint so a runaway can't create 10k pods. It cannot order, wait, or think about the fleet as a whole.
- Kueue fires in a **controller reconcile loop**, on the `Workload`, against a *cluster-wide* pool spanning many namespaces. It orders, waits, borrows, preempts.

Why `ResourceQuota` alone is insufficient for a shared GPU fleet: it has no waiting room and no cross-namespace pool. Two teams each capped at 4 GPUs of an 8-GPU fleet can never let one team burst to 8 while the other is idle — the cap rejects it, and the idle GPUs burn money. Kueue models the 8 as one pool, gives each team a floor, and lets borrowing fill the gap (06.4). Keep `ResourceQuota` as a coarse guardrail; use Kueue for the actual scheduling economics.

### New since the last pass: `AdmissionFairSharing` (beta-default, v0.15+)

Everything above governs *whether* a Workload is admitted against a fixed pool. It says nothing about *order* when several Workloads in the **same** ClusterQueue — submitted by different LocalQueues, i.e. different teams sharing one pool — are all eligible at once. That's the gap `AdmissionFairSharing` closes, and it's genuinely new depth since Kueue's earlier releases: promoted to **beta and enabled by default in Kueue v0.15**, per the feature's own design proposal (KEP-4136).

The mechanism, in outline:

- Each ClusterQueue (and, at cohort scope, each Cohort) tracks a `FairSharingStatus` per LocalQueue, with a `WeightedShare` and a `ConsumedResources` figure.
- Historical usage is tracked with a **decaying weighted average**: `new_usage = (1-A) × previous_usage + A × current_usage`, so recent consumption matters more than consumption from weeks ago, but old usage doesn't vanish instantly either.
- When multiple pending Workloads in one ClusterQueue could all fit, Kueue **admits the one whose LocalQueue has consumed the least** first — "workloads from sources that use less are admitted before workloads coming from sources that use more," in the KEP's own words.
- An **entry penalty** discourages gaming the mechanism by rapidly submitting and cancelling small jobs to look artificially "light."

The critical thing to hold in your head — because it is the exact distinction the module's brief flags as the sharpest new interview trap — is that this is a **different fairness layer** from the cohort-level `fairSharing` you'll learn in 06.4:

| | `AdmissionFairSharing` (this lesson) | `fairSharing` (06.4) |
|---|---|---|
| Scope | **within one ClusterQueue**, across its LocalQueues | **across ClusterQueues** in a cohort |
| Mechanism | admission **ordering** — who gets picked first from the pending list | **preemption** — evicting an already-admitted Workload to rebalance share |
| Question it answers | "Two teams share one pool with no borrowing involved — whose job goes next?" | "Team A is borrowing from idle Team B capacity — who gets evicted when B wants it back?" |
| Enabled since | beta-default, Kueue v0.15 | has existed longer, still requires `Configuration.fairSharing.enable: true` |

You'll see in 06.4 that a single Workload can, in principle, be affected by both: it can lose its place in the *admission* order because its LocalQueue has been consuming more than its siblings within the ClusterQueue, and separately be a preemption *target* later because its ClusterQueue is over-share within the cohort. They compose, but they are not the same knob, and conflating them is a fast way to lose credibility with an interviewer who has actually read the v0.15 release notes.

## Perspectives

**Developer.** From a researcher's seat, Kueue is nearly transparent: submit a normal `Job` with one extra label, and either it starts immediately or shows as `Pending` with a legible "waiting for quota" reason — a world away from the old `ResourceQuota` experience of a hard, confusing `403` at `kubectl apply` time that reads like a permissions bug rather than a capacity one.

**Operator.** The operator's real day-to-day job isn't writing YAML, it's **ResourceFlavor design**: how many flavors to create — per GPU SKU? per node pool? per topology zone? Too few flavors and quota can't distinguish A100 capacity from H100 capacity; too many and both the admission logic and the showback report fragment into noise nobody can read.

**Controller/systems (module 02 callback).** Kueue is a textbook reconciler: a mutating webhook forces `spec.suspend=true`, a controller watches `Workload` objects and drives a state machine (`QuotaReserved` → `Admitted` → running → complete), then flips `suspend=false` on admission. Reading `pkg/controller/jobframework` is genuinely one of the best "controller pattern in the wild" exercises available in this space — it's the same reconcile loop you studied in module 02, applied to a domain you now care about professionally.

**Economics.** The suspend-not-reject model is what makes an **oversubscribed** fleet *safe* rather than merely tolerated. Without a queue, teams either over-provision idle floors defensively (waste) or get rejected and retry-storm (also waste, plus noisy alerts). Kueue lets the platform sell 100% of nominal capacity on paper and let the queue absorb the variance — the entire reason a fixed, expensive GPU fleet can be run near its ceiling without teams silently hoarding capacity "just in case."

## Real-world use cases

- **CoreWeave — "Kueue: Kubernetes-Native AI Workload Scheduling with CoreWeave"** — https://www.coreweave.com/blog/kueue-a-kubernetes-native-system-for-ai-training-workloads. What it shows: CoreWeave's own account of running Kueue on CoreWeave Kubernetes Service for "world-class AI labs" doing both training and batch-type inference — the exact company named in this module's README running the exact tool this lesson teaches, and their own framing of why GPU training's "all-at-once" semantics need a queueing layer, not just a scheduler. (Blocked by this session's network egress proxy at fetch time; URL is search-confirmed and its content independently corroborated via a second search pass.)
- **Netflix TechBlog — "How Netflix Simplified Batch Compute with Kueue"** (June 2026) — https://netflixtechblog.com/how-netflix-simplified-batch-compute-with-kueue-87860682629c. What it shows: Kueue fully rolled out in production at Netflix, now managing **millions of batch workloads**, replacing a homegrown scheduler — Netflix deliberately migrated its *largest, most complex* customer first to stress-test the system, and the production migration took only **4 weeks** with users noticing nothing. A strong migration-strategy story, not just a technology one. (Egress-blocked at fetch time; search-confirmed.)
- **IBM Research — Vela / Blue Vela GPU supercomputers** — background paper (arXiv 2407.05467, "The infrastructure powering IBM's Gen AI model development") and the KubeCon EU 2025 tutorial "Build An AI Cluster Tutorial" (Grove/Misale/Tardieu) — https://static.sched.com/hosted_files/kccnceu2025/9b/BuildAnAIClusterTutorial.pdf. What it shows: "Vela's challenge is not about getting more GPUs, but getting more out of the GPUs they already have" — a top-tier research lab using Kueue's queueing and cohort model explicitly to solve *utilization*, not capacity, on A100 (Vela) and H100 (Blue Vela) clusters — direct validation of this lesson's central thesis. (Both egress-blocked at fetch time; search-confirmed.)
- **Kubernetes upstream — "Introducing Kueue"** (kubernetes.io/blog, Oct 2022) — https://kubernetes.io/blog/2022/10/04/introducing-kueue/. What it shows: SIG-Batch's own original design rationale — why suspend, why a controller, why not just extend `ResourceQuota` — straight from the people who built it. (Egress-blocked at fetch time; search-confirmed, and consistent with the mechanism described in the currently-fetchable Kueue GitHub source and KEPs.)

## Worked example

Fleet: 16 A100s across GPU nodes, two teams (research, product). Policy: each team owns 8 (hard floor for now, no borrowing yet). One flavor.

```yaml
# 00-flavor.yaml
apiVersion: kueue.x-k8s.io/v1beta1
kind: ResourceFlavor
metadata: { name: a100 }
spec:
  nodeLabels: { gpu-type: a100 }
---
# 01-cq-research.yaml
apiVersion: kueue.x-k8s.io/v1beta1
kind: ClusterQueue
metadata: { name: cq-research }
spec:
  namespaceSelector: {}
  resourceGroups:
  - coveredResources: ["nvidia.com/gpu"]
    flavors:
    - name: a100
      resources: [{ name: "nvidia.com/gpu", nominalQuota: 8 }]
---
# 01-cq-product.yaml  (same, name cq-product, nominalQuota 8)
---
# 02-lq.yaml
apiVersion: kueue.x-k8s.io/v1beta1
kind: LocalQueue
metadata: { namespace: research, name: lq-research }
spec: { clusterQueue: cq-research }
# + lq-product in namespace product -> cq-product
```

Team research submits three 4-GPU jobs to `lq-research`. Pool owns 8 → jobs 1 and 2 admit immediately (8 GPUs), job 3 suspends (`QuotaReserved=False`). When job 1 completes, 4 GPUs free, job 3 admits. Product's 8 GPUs were never touched — floors are isolated. That isolation is the point of 06.3; 06.4 makes the floors *lendable*.

**Showback preview.** Now extend the same window with a second LocalQueue, `lq-research-batch`, also pointed at `cq-research` (so 06.3's `AdmissionFairSharing` actually has something to order between). Suppose over a toy 1-hour window: `lq-research` submits jobs totaling 24 GPU-minutes of admitted time, `lq-research-batch` submits jobs totaling 8 GPU-minutes. If a third job from either LocalQueue arrives needing the last free slot, `AdmissionFairSharing` prefers `lq-research-batch` — it has consumed less. Sum admitted GPU-seconds **per ClusterQueue** (not per LocalQueue) over the hour and you have exactly the raw input the module's showback report (`practice/kueue-showback`) is built from: `cq-research` owed cost = (admitted GPU-seconds ÷ 3600) × $/GPU-hr, independent of which LocalQueue inside it ran the work. The LocalQueue-level fairness knob and the ClusterQueue-level cost attribution are two different axes of the same usage data — one governs who runs next, the other governs who gets billed.

## Practice (kind + fake GPUs → deliverable)

You do not need real GPUs. Advertise a **fake** `nvidia.com/gpu` on kind nodes so quota math is real while pods schedule on CPU.

```bash
# 1. Cluster
kind create cluster --name kueue-lab

# 2. Fake GPU capacity: patch node status with an extended resource.
#    (Extended resources are opaque integers the scheduler tracks.)
NODE=$(kubectl get nodes -o name | head -1 | cut -d/ -f2)
kubectl proxy --port=8001 &
curl -s --header "Content-Type: application/json-patch+json" -XPATCH \
  http://localhost:8001/api/v1/nodes/$NODE/status \
  --data '[{"op":"add","path":"/status/capacity/nvidia.com~1gpu","value":"16"}]'
kubectl label node $NODE gpu-type=a100 --overwrite

# 3. Install Kueue (check github releases for the newest tag; v0.19.0 is current as of writing)
kubectl apply --server-side -f \
  https://github.com/kubernetes-sigs/kueue/releases/download/v0.19.0/manifests.yaml
kubectl -n kueue-system wait --for=condition=Available deploy/kueue-controller-manager --timeout=120s

# 4. Namespaces + the manifests from the worked example
kubectl create ns research; kubectl create ns product
kubectl apply -f 00-flavor.yaml -f 01-cq-research.yaml -f 01-cq-product.yaml -f 02-lq.yaml

# 5. A queued Job template (4 fake GPUs, sleeps). Note: NO real GPU workload.
cat <<'EOF' > job.yaml
apiVersion: batch/v1
kind: Job
metadata: { generateName: sleeper-, namespace: research,
  labels: { kueue.x-k8s.io/queue-name: lq-research } }
spec:
  parallelism: 1
  completions: 1
  suspend: true                # Kueue enforces this anyway
  template:
    spec:
      restartPolicy: Never
      containers:
      - name: c
        image: registry.k8s.io/e2e-test-images/agnhost:2.53
        args: ["sleep","600"]
        resources: { requests: { "nvidia.com/gpu": "4" } }
EOF

# 6. Submit three; watch two admit, one suspend, then admit as quota frees
for i in 1 2 3; do kubectl create -f job.yaml; done
watch kubectl get workloads -n research \
  -o custom-columns=NAME:.metadata.name,RESERVED:.status.conditions[?\(@.type==\"QuotaReserved\"\)].status,ADMITTED:.status.admission.clusterQueue
```

Delete an admitted Job and watch the third flip to `RESERVED=True` and unsuspend.

**Stretch — see `AdmissionFairSharing` in action:** add a second LocalQueue in the `research` namespace (`lq-research-batch`) pointed at the same `cq-research`, submit uneven amounts of work to each over a short window, then submit one more job from each simultaneously when only one GPU slot is free. If your Kueue version has `AdmissionFairSharing` enabled (beta-default since v0.15 — verify with `kubectl get configuration -o yaml` or check the controller's feature-gate flags), the LocalQueue with less historical consumption should admit first. Capture the two `Workload` objects' admission order and timestamps as evidence.

**Acceptance (feeds the deliverable):**
- Kueue controller `Available` in `kueue-system`.
- One ResourceFlavor, two ClusterQueues (nominalQuota 8 each on fake `nvidia.com/gpu`), two LocalQueues, all `kubectl get` clean.
- A captured **suspend → admit** transition: a `Workload` observed at `QuotaReserved=False`, then `True`+admitted after freeing quota. Save the `kubectl get workloads` before/after and the `describe` event `couldn't assign flavors`.

Keep these manifests — 06.4 puts both ClusterQueues in a cohort and turns the hard floors into borrowable quota.

## Common pitfalls

- **Treating `nominalQuota` as if it were `ResourceQuota`.** Conceptually similar (a number), operationally different (queued, not rejected). Teams migrating from `ResourceQuota` often expect synchronous rejection and are confused when a job just sits `Pending` with `QuotaReserved=False` instead of erroring — this is one of the most common "why didn't my job run" support tickets on a freshly-Kueue-enabled cluster.
- **One ResourceFlavor per node instead of per node *shape*.** Over-fragmenting flavors makes quota math unreadable and showback reports noisy; the flavor should map to a *homogeneous class* of capacity, not literal per-node inventory.
- **Forgetting the mutating webhook requires the Kueue label at submission time.** A Job submitted without `kueue.x-k8s.io/queue-name` bypasses Kueue entirely and schedules normally through the default scheduler — silently defeating every quota, fairness, and showback guarantee the platform team believes it has. This is a dangerous silent failure on a shared fleet, not a loud error.
- **Conflating `AdmissionFairSharing` with cohort-level `fairSharing`.** They operate at different scopes (within-ClusterQueue admission order vs. across-ClusterQueue preemption) and were introduced/matured on different timelines. Using the wrong one in an explanation is the single fastest way to look like you skimmed the release notes instead of reading them.

## Self-check

- Why does Kueue suspend jobs instead of rejecting them like `ResourceQuota`? **Answer:** To preserve intent and ordering. `ResourceQuota` rejects synchronously at apiserver admission and the workload is gone — no queue, no fairness, retry storms. Suspending (via the native `Job.spec.suspend`) holds the work as an inert `Workload` object consuming zero GPUs, so Kueue can admit it later in order when quota frees. That is what lets an expensive fleet be safely oversubscribed and kept hot; a rejecter cannot queue.
- What does a ResourceFlavor map to, and why do you need it? **Answer:** It maps abstract quota to a concrete node shape via `nodeLabels` (and optional taints/tolerations) — e.g. `gpu-type: a100`. You need it because "8 GPUs" is ambiguous across heterogeneous hardware; quota is denominated *per flavor* so an A100 pool and an H100 pool are counted separately. On admission Kueue injects the flavor's `nodeSelector` into the Job so pods land on the matching nodes. No flavor → no way to tie a quota number to real capacity.
- `ResourceQuota` vs Kueue ClusterQueue quota — when does each fire, and why is `ResourceQuota` alone insufficient for a shared GPU fleet? **Answer:** `ResourceQuota` fires at kube-apiserver admission, per namespace, on committed usage, and rejects over-quota pods. Kueue fires in a controller reconcile against a cluster-wide pool and queues them. `ResourceQuota` alone can't share a GPU fleet: it has no waiting room and no cross-namespace pool, so two teams capped at half the fleet can never let one burst into the other's idle GPUs — the cap just rejects, and idle GPUs burn money. Use `ResourceQuota` as a coarse ceiling; use Kueue for scheduling economics.
- What's the practical difference between `AdmissionFairSharing` (v0.15+) and the cohort-level `fairSharing` taught in 06.4? **Answer:** `AdmissionFairSharing` operates *within one ClusterQueue*, ordering which pending Workload from which LocalQueue is admitted next based on decayed historical consumption — no preemption, no borrowing, just admission order. Cohort-level `fairSharing` (06.4) operates *across ClusterQueues* in a cohort and can trigger preemption of already-admitted Workloads to rebalance dominant-resource share. A single Workload can be affected by both at once: deprioritized in admission order by `AdmissionFairSharing`, and later a preemption target under cohort `fairSharing` if its ClusterQueue is borrowing heavily.
- A Job is submitted without the `kueue.x-k8s.io/queue-name` label. What happens to it, and why is that dangerous on a shared fleet? **Answer:** Kueue's mutating webhook never touches it, so it is never suspended, never gets a `Workload`, and schedules immediately through the default scheduler as if Kueue didn't exist — silently bypassing every quota, ordering, and fairness guarantee the platform believes is in force. It's dangerous precisely because it fails silently: no error, no rejected admission, just a job quietly consuming un-tracked GPUs outside the accounting system the showback report depends on.

## Connections & what's next

06.3 gave every team a hard, isolated floor — safe, but wasteful the moment one team is idle and another is queued. That waste is exactly the "held-but-idle GPUs = pure burn" line from the module README, and it sets up the next lesson directly: 06.4 turns these same ClusterQueues into a **cohort**, where `nominalQuota` becomes a floor you can temporarily exceed by borrowing from an idle sibling, and where preemption reclaims that borrowed capacity the moment the owner needs it back. The `AdmissionFairSharing` mechanism introduced here also resurfaces there — you'll see explicitly how it interacts with (and differs from) cohort-level `fairSharing`, and why Kueue's fairness math for both ultimately rests on the same theoretical foundation: Dominant Resource Fairness.

## References & further reading

**Primary sources**
- Kueue ClusterQueue concept — https://kueue.sigs.k8s.io/docs/concepts/cluster_queue/ — read for `resourceGroups`, flavors, `nominalQuota` semantics in full. (Egress-blocked at fetch time; search-confirmed canonical doc URL.)
- Kueue concepts index and Installation task — https://kueue.sigs.k8s.io/docs/concepts/ · https://kueue.sigs.k8s.io/docs/getting-started/installation/ — read for ResourceFlavor, LocalQueue, and Workload object references, and the install procedure. (Egress-blocked at fetch time; search-confirmed.)
- Kueue Admission Fair Sharing concept — https://kueue.sigs.k8s.io/docs/concepts/admission_fair_sharing/ — read for the shipped, documented behavior of the mechanism covered above. (Egress-blocked at fetch time; URL confirmed to exist via search.)
- KEP-4136, Admission Fair Sharing — https://github.com/kubernetes-sigs/kueue/tree/main/keps/4136-admission-fair-sharing — read for the full design: `FairSharingStatus`, the decayed-usage formula, entry penalties, and the cohort-scope mixed-mode admission rules. **Fetched directly** this session — confirmed content matches the summary above.
- Kubernetes blog, "Introducing Kueue" (Oct 2022) — https://kubernetes.io/blog/2022/10/04/introducing-kueue/ — read for SIG-Batch's own original rationale: why suspend, why a controller, why not extend `ResourceQuota`. (Egress-blocked at fetch time; search-confirmed.)
- Kueue source — https://github.com/kubernetes-sigs/kueue — read `pkg/controller/jobframework` (the suspend webhook) and `pkg/scheduler` as a Go controller-reading exercise, tying back to module 02.

**Real-world engineering blogs**
- CoreWeave — "Kueue: Kubernetes-Native AI Workload Scheduling with CoreWeave" — https://www.coreweave.com/blog/kueue-a-kubernetes-native-system-for-ai-training-workloads — what it shows: a named target-employer running Kueue in production for AI-lab customers.
- Netflix TechBlog — "How Netflix Simplified Batch Compute with Kueue" — https://netflixtechblog.com/how-netflix-simplified-batch-compute-with-kueue-87860682629c — what it shows: Kueue at millions-of-workloads scale, plus a real migration-sequencing strategy (hardest customer first).
- IBM Research — Vela/Blue Vela — arXiv 2407.05467 and https://static.sched.com/hosted_files/kccnceu2025/9b/BuildAnAIClusterTutorial.pdf — what it shows: a research supercomputer using Kueue's queueing/cohort model explicitly to solve utilization, not capacity.
- CERN/WLCG — "WLCG OTF: Kueue Overview and Upcoming Enhancements" — https://indico.cern.ch/event/1552799/contributions/6569144/attachments/3092053/5476708/WLCG%20OTF_%20Kueue%20Overview%20and%20Upcoming%20Enhancements.pdf — what it shows: a non-hyperscaler, scientific-computing adopter evaluating Kueue for grid-computing batch workloads — evidence this isn't only a hyperscaler pattern. (Egress-blocked at fetch time; URL confirmed via search.)

**Deeper dives**
- Kueue release notes / GitHub Releases — https://github.com/kubernetes-sigs/kueue/releases — check for the current stable tag before installing; the module README's "verify feature gates" warning applies directly to `AdmissionFairSharing` and TAS.
- Kueue GitHub issues — https://github.com/kubernetes-sigs/kueue/issues — browse open issues for live design discussion (see 06.4's reference to issue #7016 on cohort borrowing order) — a good way to see how these semantics are still evolving in the open.
