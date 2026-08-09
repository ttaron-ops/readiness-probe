---
lesson: "06.3"
title: "Kueue's queueing model: suspend, admit, and the quota pool"
module: "06"
concept: "Kueue's queueing model: suspend, admit, and the quota pool"
status: not-started
est_time: "8h"
artifacts: []
---
# 06.3 · Kueue's queueing model: suspend, admit, and the quota pool
> **Concept.** Kueue admits work by *suspending* Jobs and *unsuspending* them when quota is free — a queue, not a wall.
>
> Module: [🗓️ 06 — Scheduling, queueing and capacity](../README.md) · Deliverable: [Kueue setup + showback](../practice/kueue-showback/README.md)

## Why this matters

You run 40+ clusters. On a GPU fleet the scarce unit is not pods — it is `nvidia.com/gpu`, and it is contended by many teams. The default Kubernetes answer to "team X wants more than its share" is `ResourceQuota`: the API server rejects the over-quota pod **synchronously, at create time**, with a `403 exceeded quota`. The workload is gone. Nothing remembers it wanted to run. There is no ordering, no fairness, no "run it when capacity frees." For a shared, expensive, always-oversubscribed GPU fleet that is the wrong primitive: you want a **queue** that holds the work and admits it in order as GPUs free up, so the fleet stays hot and teams get FIFO-ish fairness instead of a retry storm.

That is Kueue. CoreWeave runs it in production; Anthropic names it in platform JDs. It is *the* batch-admission layer for GPU Kubernetes, and the ClusterQueue is the unit you attribute cost against in showback. This lesson is half the anchor: the queueing model. The next (06.4) is borrowing and preemption across cohorts — the deep interview surface.

## What's new here (vs 02 and 04)

- **vs 04 (`ResourceQuota`, GPU quotas):** `ResourceQuota` is an *admission-control rejecter*. It counts committed usage in a namespace and **rejects** the create request when a pod would exceed the cap. It is synchronous, per-namespace, and stateless about intent — rejected work vanishes. Kueue's ClusterQueue is an *admission-control queuer*: over-quota work is **held (suspended) and remembered**, then admitted later. Different verbs: **reject vs queue.** They also live at different layers — `ResourceQuota` is enforced by the kube-apiserver; Kueue is a controller reconciling a `Workload` CRD.
- **vs 02 (controllers, informers, CRDs):** everything here is the controller pattern you already know. Kueue watches Jobs via a webhook + informers, materialises a `Workload` custom resource per Job, and a reconciler drives it through states. It is a clean, well-factored Go controller — a good source-reading exercise (see Resources). You are not learning a new paradigm, you are learning a new set of CRDs and one clever mechanism: **suspend**.

## Core notes

### The mechanism: `suspend`

Kubernetes `Job` has a native field `spec.suspend` (bool, GA since 1.24). When `true`, the Job controller creates **no pods** and terminates any that exist; the Job sits inert. Kueue weaponises this:

1. You submit a Job carrying the label `kueue.x-k8s.io/queue-name: <local-queue>`.
2. Kueue's mutating webhook **forces `spec.suspend: true`** on admission of the Job object (even if you set it false). No pods are created. No GPUs are consumed.
3. Kueue creates a `Workload` object that represents the Job's total resource ask (pods × per-pod requests).
4. The Kueue scheduler evaluates the `Workload` against its LocalQueue → ClusterQueue quota. If quota is available in some ResourceFlavor, it **reserves quota** and writes an admission onto the `Workload`.
5. On admission Kueue **sets `spec.suspend: false`** and injects the flavor's `nodeSelector`/tolerations. *Now* the Job controller creates pods, which the kube-scheduler binds to matching nodes.
6. On completion, quota is released; the next queued `Workload` is admitted.

That is the whole model. Rejection never happens for capacity reasons — the `Workload` waits with condition `QuotaReserved=false` and event `couldn't assign flavors`. This is the answer to "why suspend not reject": **suspend preserves intent and ordering so the fleet can be oversubscribed safely.**

### The four objects

| Object | Scope | Role | Who owns it |
|---|---|---|---|
| **ResourceFlavor** | cluster | Maps abstract quota to a *node shape* via labels/taints (e.g. `gpu=a100`) | platform |
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

This indirection is the RBAC/multi-tenancy seam: teams get `create` on Jobs and see their LocalQueue; only platform edits ClusterQueues and Flavors. It is also the showback seam — usage rolls up per ClusterQueue.

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

# 3. Install Kueue (check github releases for the newest tag; v0.19.0 shown)
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

**Acceptance (feeds the deliverable):**
- Kueue controller `Available` in `kueue-system`.
- One ResourceFlavor, two ClusterQueues (nominalQuota 8 each on fake `nvidia.com/gpu`), two LocalQueues, all `kubectl get` clean.
- A captured **suspend → admit** transition: a `Workload` observed at `QuotaReserved=False`, then `True`+admitted after freeing quota. Save the `kubectl get workloads` before/after and the `describe` event `couldn't assign flavors`.

Keep these manifests — 06.4 puts both ClusterQueues in a cohort and turns the hard floors into borrowable quota.

## Self-check

**(a) Why does Kueue SUSPEND jobs instead of rejecting them like `ResourceQuota`?**
**Answer:** To preserve intent and ordering. `ResourceQuota` rejects synchronously at apiserver admission and the workload is gone — no queue, no fairness, retry storms. Suspending (via the native `Job.spec.suspend`) holds the work as an inert `Workload` object consuming zero GPUs, so Kueue can admit it later in order when quota frees. That is what lets an expensive fleet be safely oversubscribed and kept hot; a rejecter cannot queue.

**(b) What does a ResourceFlavor map to, and why do you need it?**
**Answer:** It maps abstract quota to a concrete node shape via `nodeLabels` (and optional taints/tolerations) — e.g. `gpu-type: a100`. You need it because "8 GPUs" is ambiguous across heterogeneous hardware; quota is denominated *per flavor* so an A100 pool and an H100 pool are counted separately. On admission Kueue injects the flavor's `nodeSelector` into the Job so pods land on the matching nodes. No flavor → no way to tie a quota number to real capacity.

**(c) k8s `ResourceQuota` vs Kueue ClusterQueue quota — when does each fire, and why is `ResourceQuota` alone insufficient for a shared GPU fleet?**
**Answer:** `ResourceQuota` fires at kube-apiserver admission, per namespace, on committed usage, and rejects over-quota pods. Kueue fires in a controller reconcile against a cluster-wide pool and queues them. `ResourceQuota` alone can't share a GPU fleet: it has no waiting room and no cross-namespace pool, so two teams capped at half the fleet can never let one burst into the other's idle GPUs — the cap just rejects, and idle GPUs burn money. Use `ResourceQuota` as a coarse ceiling; use Kueue for scheduling economics.

## Resources

1. **Kueue ClusterQueue concept** — the deep reference for `resourceGroups`, flavors, `nominalQuota`. https://kueue.sigs.k8s.io/docs/concepts/cluster_queue/
2. **Kueue concepts index** (ResourceFlavor, LocalQueue, Workload) + **Installation task**. https://kueue.sigs.k8s.io/docs/concepts/ · https://kueue.sigs.k8s.io/docs/getting-started/installation/
3. **Kueue source** — read `pkg/controller/jobframework` (the suspend webhook) and `pkg/scheduler` as a Go controller-reading exercise (ties back to module 01/02). https://github.com/kubernetes-sigs/kueue
