# External depth library — `harut8/system-design`

> **What this is.** A curated bridge from *this* course to
> [`harut8/system-design`](https://github.com/harut8/system-design)
> ([read online](https://harut8.github.io/system-design/)) — a ~400,000-line set of
> systems-engineering notes covering CPU/OS internals, storage engines, Kubernetes internals,
> SRE observability, GPU observability, and RAG.
>
> **Why it earns a place here.** This course is a *build* curriculum: every lesson is short on
> purpose and gates on an artifact. That design deliberately leaves the "read the 2,500-line
> internals chapter" slot empty. `harut8/system-design` fills exactly that slot, at a depth this
> course does not attempt and should not try to duplicate.

## How to use it (the one rule)

**Depth reading is a lesson input, never a lesson substitute.** The course's operating principle
is *build, do not read* — and the source repo agrees with it in its own words:

> "If you are reading something that isn't unblocking a build, stop and go build."
> — [`ai-rag/README.md`](https://github.com/harut8/system-design/blob/main/ai-rag/README.md)

So: open a depth chapter **when a lesson's practice artifact is blocked on internals you don't
have**, not as a warm-up. A lesson is still only `done` when its artifact exists.

## Attribution and licensing

The source repository carries **no LICENSE file**, which means default copyright — all rights
reserved. Nothing has been copied into this course. Everything here is:

- **links** to the source chapters, plus
- **our own summaries** of what each chapter is good for, plus
- **material rewritten and re-aimed** at this course's GPU-platform target (clearly marked, with
  the source credited at the top of the file).

If you ever want to lift text verbatim — for a blog post, a portfolio doc, anything public —
ask the author first, or cite and link rather than quote at length. The author states in the repo
README that these are personal study notes and that corrections are welcome via issue or PR.

**Snapshot:** commit `9bcf6bf`, read 2026-08-15. Chapter numbering below matches that snapshot;
the repo is actively written, so file names may drift.

---

## The eight tracks, and what each is worth to you

| Track | Chapters | Value to this course |
|---|---:|---|
| [`kubernetes/`](https://github.com/harut8/system-design/tree/main/kubernetes) | 46 | **Highest.** 2,300–3,400 lines per chapter on etcd, apiserver, scheduler, kubelet, CNI/Cilium, CRDs. This is the internals layer under Modules 02, 04, 06, 10 and `platform/02`. |
| [`sre-observability/`](https://github.com/harut8/system-design/tree/main/sre-observability) | 47 | **High.** Storage internals per signal, cardinality economics, SLO engineering. Directly under `platform/03`. |
| [`gpu-observability/`](https://github.com/harut8/system-design/tree/main/gpu-observability) | 23 | **High, near-1:1 with Module 05.** Mostly confirms your coverage; adds the DCGM field-ID cheat sheet, troubleshooting flowcharts, and the lakehouse chapter. |
| [`databases/`](https://github.com/harut8/system-design/tree/main/databases) | 24 | **Medium-high.** WAL, LSM, B-tree, fsync, leader election. The durability layer under etcd ops (M10) and checkpoint stores (M08). |
| [`k8s-learn/`](https://github.com/harut8/system-design/tree/main/k8s-learn) | 14 task sheets | **High — practice, not reading.** Hands-on task ladders. `gpu-platform-tasks.md` is the best single file in the repo for this course (see the lab it seeded, below). |
| [`python-mastery/`](https://github.com/harut8/system-design/tree/main/python-mastery) | 30 | **Mixed.** Chapters 00–12 are language-agnostic OS/hardware internals and are excellent for Module 01b. The CPython half is off-target for a Go-first platform role. |
| [`tasks/` · `solutions/` · `implementation/`](https://github.com/harut8/system-design/tree/main/solutions) | 4 problems × 4 scale tiers | **High for interview prep.** Worked design docs at 10k → 10m scale. Seeded the design-drill ladder in `platform/01`. |
| [`ai-rag/`](https://github.com/harut8/system-design/tree/main/ai-rag) | 9 + labs | **Optional / adjacent.** Out of scope for a GPU-platform role, but see the note under Module 07 if you target inference-platform roles that own retrieval. |

Plus [`SYSTEM-DESIGN-GUIDE.md`](https://github.com/harut8/system-design/blob/main/SYSTEM-DESIGN-GUIDE.md)
— a 937-line interview framework (requirements → estimation → API → high-level → deep dive →
tradeoffs) that pairs with `platform/01` lesson 08.

---

## What this repo gave the course

Three things were genuinely additive and have been built into the course itself:

1. **[The no-hardware GPU fleet lab](../modules/04-gpu-on-kubernetes/practice/fake-gpu-fleet/)** —
   fake extended resources + `kwok` fleet + a synthetic DCGM exporter, so Modules 04/05/06/11 and
   the capstone are buildable **without renting a GPU**. Adapted from
   [`k8s-learn/gpu-platform-tasks.md`](https://github.com/harut8/system-design/blob/main/k8s-learn/gpu-platform-tasks.md).
   This removes the single largest practical blocker in the course.
2. **[`platform/03` lesson 10 — the telemetry lakehouse](../platform/03-observability/lessons/10-telemetry-lakehouse.md)** —
   a real gap. The course had no story for "what did team X's $/useful-GPU-hour do last quarter,"
   which is a FinOps question PromQL structurally cannot answer.
3. **[The design-drill ladder](../platform/01-distributed-systems-and-design/practice/staff-design-portfolio/design-drills.md)** —
   the same problem re-solved at 10k / 1m / 10m scale, which is how the source's `implementation/`
   tree is organised and a better rehearsal method than one-shot prompts.

Everything else is depth reading, mapped per module below.

---

## Per-module depth maps

Each module now carries its own `resources/depth-map.md` with **lesson-level** pointers. This
table is the index.

### Track B — GPU specialization

| Module | Depth map | Densest source material |
|---|---|---|
| [01 Go for infra](../modules/01-go-for-infra/resources/depth-map.md) | ↗ | *thin* — the source is Python-first; use `k8s-learn/controller-tasks.md` for client-go practice |
| [01b Linux internals](../modules/01b-linux-internals/resources/depth-map.md) | ↗ | `python-mastery/06–12` (processes, virtual memory, allocators, syscalls, IPC, observing a process) + `kubernetes/00` |
| [02 K8s controllers](../modules/02-kubernetes-controllers/resources/depth-map.md) | ↗ | `kubernetes/04, 05, 08, 23, 24, 36` — etcd, apiserver, client-go, controller-runtime, aggregation, GC |
| [02b Host topology](../modules/02b-host-topology/resources/depth-map.md) | ↗ | `kubernetes/10, 21` + `python-mastery/01, 07` (NUMA, cache, page tables) |
| [03 GPU hardware](../modules/03-gpu-hardware/resources/depth-map.md) | ↗ | `gpu-observability/00` (how GPUs work, for observability people) |
| [04 GPU on Kubernetes](../modules/04-gpu-on-kubernetes/resources/depth-map.md) | ↗ | `kubernetes/10` (device manager), `19` (CSI), `21` (QoS), `k8s-learn/gpu-platform-tasks.md` |
| [05 GPU observability](../modules/05-gpu-observability/resources/depth-map.md) | ↗ | the whole `gpu-observability/` track — near-1:1, plus appendices B (field IDs) and C (flowcharts) |
| [06 Scheduling & capacity](../modules/06-scheduling-capacity/resources/depth-map.md) | ↗ | `kubernetes/09, 34, 22, 35` — scheduler internals, scheduler framework, autoscaling, 15k-node tuning |
| [07 Inference serving](../modules/07-inference-serving/resources/depth-map.md) | ↗ | `gpu-observability/14`, `sre-observability/26`, `ai-rag/` (adjacent) |
| [08 Distributed training](../modules/08-distributed-training/resources/depth-map.md) | ↗ | `gpu-observability/15`, `databases/14` (WAL — the checkpoint-durability analogue) |
| [09 Networking & topology](../modules/09-networking-topology/resources/depth-map.md) | ↗ | `kubernetes/15, 16` (CNI, Cilium/eBPF), `sre-observability/24` |
| [10 Bare metal & lifecycle](../modules/10-bare-metal-lifecycle/resources/depth-map.md) | ↗ | `kubernetes/04, 32, 33, 37` + `databases/12, 14, 16` (replication, WAL, leader election) |
| [11 GPU cost economics](../modules/11-gpu-cost-economics/resources/depth-map.md) | ↗ | `gpu-observability/12, 17`, `sre-observability/18, 31, 39` |
| [12 Capstone & interview](../modules/12-capstone-interview/resources/depth-map.md) | ↗ | `SYSTEM-DESIGN-GUIDE.md`, `tasks/`, `solutions/`, `implementation/`, `kubernetes/38` |

### Track A — Platform excellence

| Module | Depth map | Densest source material |
|---|---|---|
| [01 Distributed systems & design](../platform/01-distributed-systems-and-design/resources/depth-map.md) | ↗ | `databases/05, 12, 13, 14, 16, 19` + `distributed-systems/README.md` + `SYSTEM-DESIGN-GUIDE.md` |
| [02 Platform networking](../platform/02-platform-networking/resources/depth-map.md) | ↗ | `kubernetes/14–18, 20` — kube-proxy, CNI, Cilium/eBPF, Gateway/mesh, DNS, NetworkPolicy |
| [03 Observability](../platform/03-observability/resources/depth-map.md) | ↗ | the whole `sre-observability/` track — 47 chapters, the deepest single match in the repo |

---

## Honest limits of the source

Worth knowing before you lean on it:

- **No license** — see above. Link, don't lift.
- **Personal study notes**, self-described as carrying "the occasional half-finished section."
  `distributed-systems/` is a roadmap only; `ai-rag/` is missing chapters 05–07 and 09–14 that its
  own index promises.
- **Two files collide** — `databases/11-hnsw-vector-search-internals.md` and
  `databases/11-vector-search-internals.md` share an index number.
- **Python-first where this course is Go-first.** The `python-mastery/` CPython chapters
  (15–43) are excellent but off your path; the OS/hardware chapters (00–12) are not.
- **Not GPU-fleet-economics-first.** Its GPU material is observability-shaped; the cost,
  commitment, and unit-economics thinking in Module 11 has no counterpart there. That remains
  your differentiator, not something to import.

## References

- Repository — https://github.com/harut8/system-design
- Published site — https://harut8.github.io/system-design/ *(egress-blocked from this
  environment; read the Markdown in the repo instead — it is the same content)*
