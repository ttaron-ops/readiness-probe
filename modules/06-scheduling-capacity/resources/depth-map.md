# Depth map — Module 06 · Scheduling & capacity

Pointers into [`harut8/system-design`](https://github.com/harut8/system-design). **Open a chapter
only when a lesson's artifact is blocked on internals you don't have** — see
[`docs/EXTERNAL-DEPTH.md`](../../../docs/EXTERNAL-DEPTH.md) for how to use this library and the
attribution/licensing note.

> **Strong match on mechanism, weak on economics.** The source has two very deep scheduler
> chapters (2,500–2,600 lines) that go further than this module on *how the scheduler works*. It
> has nothing on fragmentation cost, commitment laddering, or capacity economics — that half is
> yours.
>
> Everything below is runnable against the
> [fake GPU fleet](../../04-gpu-on-kubernetes/practice/fake-gpu-fleet/README.md), which is where
> fragmentation and gang deadlock actually become visible.

| Lesson | Go deeper in | Why |
|---|---|---|
| 01 Why the default scheduler fails | [`kubernetes/09-kube-scheduler-internals`](https://github.com/harut8/system-design/blob/main/kubernetes/09-kube-scheduler-internals.md) | the scheduling cycle, the scheduling queue (active/backoff/unschedulable), and exactly where a Pending GPU pod sits |
| 02 Gang / co-scheduling | [`kubernetes/34-custom-schedulers-and-scheduler-framework`](https://github.com/harut8/system-design/blob/main/kubernetes/34-custom-schedulers-and-scheduler-framework.md) | the `Permit` plugin and waiting-pod mechanics — how all-or-nothing admission is actually implemented |
| 03–04 Kueue I & II | [`kubernetes/09-kube-scheduler-internals`](https://github.com/harut8/system-design/blob/main/kubernetes/09-kube-scheduler-internals.md) | preemption and the nominated-node machinery Kueue's eviction sits above |
| 03–04 Kueue I & II | [`kubernetes/25-multi-tenancy`](https://github.com/harut8/system-design/blob/main/kubernetes/25-multi-tenancy.md) | quota models and tenancy boundaries — the layer ClusterQueue cohorts formalise |
| 05 Alternatives (Volcano, KAI) | [`kubernetes/34-custom-schedulers-and-scheduler-framework`](https://github.com/harut8/system-design/blob/main/kubernetes/34-custom-schedulers-and-scheduler-framework.md) | running a second scheduler safely, and the extension points each alternative overrides |
| 06 Topology-aware placement | [`kubernetes/10-kubelet-internals`](https://github.com/harut8/system-design/blob/main/kubernetes/10-kubelet-internals.md) | the Topology Manager — the node-side half; the scheduler can only propose what the kubelet can align |
| 07 Fragmentation & effective capacity | [`kubernetes/22-autoscaling`](https://github.com/harut8/system-design/blob/main/kubernetes/22-autoscaling.md) | Cluster Autoscaler and Karpenter bin-packing and consolidation — the mechanisms that do and don't apply to GPUs |
| 08 Priority/preemption & capacity economics | [`kubernetes/35-performance-scaling-and-tuning`](https://github.com/harut8/system-design/blob/main/kubernetes/35-performance-scaling-and-tuning.md) | Kubernetes at 5,000–15,000 nodes — scheduler throughput, `percentageOfNodesToScore`, and what breaks first |

## Practice worth stealing

- [`k8s-learn/scheduling-constraints-tasks`](https://github.com/harut8/system-design/blob/main/k8s-learn/scheduling-constraints-tasks.md)
  — affinity, taints, topology spread, worked as a ladder. Good warm-up on the fake fleet.
- [`k8s-learn/gpu-platform-tasks`](https://github.com/harut8/system-design/blob/main/k8s-learn/gpu-platform-tasks.md),
  **Projects 4 and 5** — build a GPU job queue, then a scheduler-framework plugin, and *benchmark
  it against the default on a replayed trace*. Its framing is the one to adopt: **"a scheduler
  without a benchmark is an opinion."** A measured comparison on a published trace is the most
  credible artifact this module can produce.

## Where this course goes further

Fragmentation-as-stranded-capital, the effective-capacity model, commitment laddering, and why you
can't autoscale GPUs the way you autoscale CPUs. The source treats scheduling as a correctness and
throughput problem; this module treats it as a capital-efficiency problem. Keep that framing — it's
the differentiated one.
