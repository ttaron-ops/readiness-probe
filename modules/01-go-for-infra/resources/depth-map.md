# Depth map — Module 01 · Go for infra

Pointers into [`harut8/system-design`](https://github.com/harut8/system-design). **Open a chapter
only when a lesson's artifact is blocked on internals you don't have** — see
[`docs/EXTERNAL-DEPTH.md`](../../../docs/EXTERNAL-DEPTH.md) for how to use this library and the
attribution/licensing note.

> **Thin module.** The source repo is Python-first and has no Go track. Its value here is
> *practice*, not reading — and the concurrency chapters, which are language-agnostic theory that
> Go's memory model obeys too.

| Lesson | Go deeper in | Why |
|---|---|---|
| 04 Concurrency & context | [`python-mastery/02-atomics-and-memory-models`](https://github.com/harut8/system-design/blob/main/python-mastery/02-atomics-and-memory-models.md) | what a store actually promises — the hardware layer under Go's happens-before rules and `sync/atomic` |
| 04 Concurrency & context | [`python-mastery/30-concurrency-correctness`](https://github.com/harut8/system-design/blob/main/python-mastery/30-concurrency-correctness.md) | how to *test* concurrent code for correctness, which transfers directly to `-race` and deterministic controller tests |
| 05 Testing | [`python-mastery/43-testing-strategy`](https://github.com/harut8/system-design/blob/main/python-mastery/43-testing-strategy.md) | "suites that find bugs, not suites that pass" — the framing to apply to envtest |
| 09 Controller primer | [`k8s-learn/controller-tasks`](https://github.com/harut8/system-design/blob/main/k8s-learn/controller-tasks.md) | **practice, not reading** — a graded ladder of client-go tasks, writing a controller by hand before reaching for a framework |

## Practice worth stealing

[`k8s-learn/controller-tasks.md`](https://github.com/harut8/system-design/blob/main/k8s-learn/controller-tasks.md)
and [`k8s-learn/api-machinery-tasks.md`](https://github.com/harut8/system-design/blob/main/k8s-learn/api-machinery-tasks.md)
are the best on-ramp to Lesson 09 and Module 02 — they force you through informers, workqueues and
resync semantics by hand, which is exactly the gap between "I've used controller-runtime" and "I
know what it's doing."

## Deliberately skipped

The whole CPython half of `python-mastery/` (chapters 15–43 on refcounting, the eval loop, the GIL,
free-threading, asyncio internals). Excellent material, wrong language for a Go-first platform
role. Read it if you end up owning a Python inference service; not before.
