# Depth map — Module 08 · Distributed training

Pointers into [`harut8/system-design`](https://github.com/harut8/system-design). **Open a chapter
only when a lesson's artifact is blocked on internals you don't have** — see
[`docs/EXTERNAL-DEPTH.md`](../../../docs/EXTERNAL-DEPTH.md) for how to use this library and the
attribution/licensing note.

> **Sparse match, one non-obvious win.** The source has no parallelism-strategies or NCCL-internals
> material. What it does have is a **1,900-line chapter on write-ahead logging** that is the right
> mental model for checkpoint durability — a framing this module can borrow wholesale.

| Lesson | Go deeper in | Why |
|---|---|---|
| 03 Communication bottleneck | [`kubernetes/15-cni-and-pod-networking`](https://github.com/harut8/system-design/blob/main/kubernetes/15-cni-and-pod-networking.md) | the pod-network path collectives traverse when they aren't on RDMA |
| 04 Checkpointing | [`databases/14-write-ahead-log-internals`](https://github.com/harut8/system-design/blob/main/databases/14-write-ahead-log-internals.md) | **the borrowed framing.** A checkpoint is a WAL problem: durability vs write latency, `fsync` semantics, group commit, and recovery time. The tradeoff axis is identical, and the vocabulary is more precise than "how often do we checkpoint". |
| 04 Checkpointing | [`databases/01-storage-engine-fundamentals`](https://github.com/harut8/system-design/blob/main/databases/01-storage-engine-fundamentals.md) | page cache, `fsync`, and durability guarantees — what "the checkpoint is written" actually means |
| 05 Failure & elasticity | [`databases/16-failure-detection-and-leader-election`](https://github.com/harut8/system-design/blob/main/databases/16-failure-detection-and-leader-election.md) | phi-accrual detectors, timeouts, and the fail-stop vs fail-slow distinction that decides whether elastic training helps |
| 05 Failure & elasticity | [`gpu-observability/15-distributed-training-observability`](https://github.com/harut8/system-design/blob/main/gpu-observability/15-distributed-training-observability.md) | detecting a straggler rather than a corpse — the signal side of elasticity |
| 06 Job orchestration | [`kubernetes/12-workload-controllers`](https://github.com/harut8/system-design/blob/main/kubernetes/12-workload-controllers.md) · [`13-statefulset-deep-dive`](https://github.com/harut8/system-design/blob/main/kubernetes/13-statefulset-deep-dive.md) | Job/CronJob semantics and stable identity — what a training operator is built on |
| 07 Data pipeline | [`python-mastery/09-syscalls-and-io`](https://github.com/harut8/system-design/blob/main/python-mastery/09-syscalls-and-io.md) | the I/O path a dataloader stalls on |
| 07 Data pipeline | [`python-mastery/11-ipc-and-shared-memory`](https://github.com/harut8/system-design/blob/main/python-mastery/11-ipc-and-shared-memory.md) | shared-memory cost between workers — the `/dev/shm` failure every PyTorch DataLoader eventually hits |
| 08 Training economics | [`gpu-observability/15-distributed-training-observability`](https://github.com/harut8/system-design/blob/main/gpu-observability/15-distributed-training-observability.md) | goodput measurement, which is the numerator of every economics claim here |

## The contrast rep worth doing

Read [`databases/14-write-ahead-log-internals`](https://github.com/harut8/system-design/blob/main/databases/14-write-ahead-log-internals.md)
and then write the checkpoint-store design twice — once for restart RTO, once for zero-loss
durability — as the contrast rep in
[`platform/01`'s design drills](../../../platform/01-distributed-systems-and-design/practice/staff-design-portfolio/design-drills.md).
The WAL vocabulary (group commit, `fsync` batching, recovery time as a first-class requirement)
makes the tradeoff axis much easier to name out loud, which is exactly what the checkpoint tests.

## Nothing there for

Parallelism strategies (DP/TP/PP/FSDP), NCCL collectives and ring/tree algorithms, Young/Daly
checkpoint-interval math. Unique to this course.
