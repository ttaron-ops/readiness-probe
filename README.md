# 🎓 Readiness Probe — Senior GPU-Platform Engineer study repo

Working repository for a **12–15 month** preparation to land **Senior Platform
Engineer roles at companies running GPU fleets**. Built for **~10–12 focused
hrs/week**. Notion holds the original GPU study plan and progress tracking; **this
repo holds the work** — notes, lessons, code, benchmarks, and checkpoint answers.

> **Target:** Senior Platform Engineer where GPU fleet operations are mandatory.
> **Timeline:** 12–15 months at ~10–12 hrs/week.
> **Operating principle:** *Build, do not read.* Every module ends in something you
> built; the checkpoints — not the reading — are the completion gate. Evidence is
> public-by-default.

## The three tracks

The course runs three tracks **in parallel**, not one linear list.

### Track A — Platform excellence  →  `platform-eng/platform/`
Refresh and deepen core platform engineering to a senior/staff bar. Only the two—
now three—genuinely high-leverage areas are first-class **deepen modules**;
everything you already operate daily is a light **interview refresh**.

| Module | Lessons | |
|--------|:---:|---|
| 🧩 [Distributed systems & system design](platform-eng/platform/01-distributed-systems-and-design/README.md) | 9 | deepen |
| 🌐 [Platform networking depth](platform-eng/platform/02-platform-networking/README.md) | 8 | deepen |
| 🔭 [Observability engineering](platform-eng/platform/03-observability/README.md) | 9 | deepen |
| 🔁 [Interview refresh](platform-eng/platform/interview-refresh/README.md) | 6 pages | light — IaC · CI/CD & GitOps · cloud/multi-cloud · security · SRE · k8s ops |

### Track B — GPU specialization  →  `platform-eng/modules/`
The gap-closing path, mapped 1:1 to the [Notion study plan](https://app.notion.com/p/3b23abaeb82380beb8a1d5dd935b9fd0).

**Core:** [01 🐹 Go](platform-eng/modules/01-go-for-infra/README.md) · [01b 🐧 Linux internals](platform-eng/modules/01b-linux-internals/README.md) · [02 ⚙️ K8s controllers](platform-eng/modules/02-kubernetes-controllers/README.md) · [02b 🧬 Host topology](platform-eng/modules/02b-host-topology/README.md) · [03 🔌 GPU hardware](platform-eng/modules/03-gpu-hardware/README.md) · [04 📦 GPU on Kubernetes](platform-eng/modules/04-gpu-on-kubernetes/README.md) · [05 📊 GPU observability](platform-eng/modules/05-gpu-observability/README.md) · [06 🗓️ Scheduling & capacity](platform-eng/modules/06-scheduling-capacity/README.md) · [07 🚀 Inference serving](platform-eng/modules/07-inference-serving/README.md) · [11 💰 GPU cost & unit economics](platform-eng/modules/11-gpu-cost-economics/README.md)

**Stretch (deferrable — learn on the job or for a specific employer):** [08 🧮 Distributed training](platform-eng/modules/08-distributed-training/README.md) · [09 🔗 Networking & topology](platform-eng/modules/09-networking-topology/README.md) · [10 🖥️ Bare metal & lifecycle](platform-eng/modules/10-bare-metal-lifecycle/README.md)

### Track C — Evidence & interview  →  [`modules/12-capstone-interview`](platform-eng/modules/12-capstone-interview/README.md)
The public proof that converts knowledge into offers.
- **Flagship:** GPU cost/efficiency controller — Stage 1 Metrics MVP → Stage 2 CRDs → Stage 3 fractional attribution
- **Writing:** the "your GPU dashboard is lying" post + two more
- **Benchmark:** cost-per-million-tokens curve (a vLLM weekend)
- **Interview:** system-design track + behavioral / staff-signal stories

## The plan

The month-by-month calendar, milestones, and readiness gate live in
**[`docs/ROADMAP.md`](docs/ROADMAP.md)**. Summary:

| Phase | Months | Focus |
|-------|--------|-------|
| 0 · Foundations | 1–3 | B: Go + Linux + start controllers · A1 distributed systems begins · line up the open-source conversation |
| 1 · Controller + GPU base | 3–6 | B: finish controllers, host topology, GPU hardware, start GPU-on-k8s · **C: Metrics MVP (~m4)** · A2 networking |
| 2 · GPU depth + evidence | 6–9 | B: observability, scheduling, inference · A3 observability (pairs with B05) · **C: CRDs, benchmark, 2 posts** |
| 3 · Cost + interview ramp | 9–12 | B: cost module · **C: fractional attribution, mocks, behavioral, post #3** · refresh sharp areas · **start interviewing** |
| 4 · Close | 12–15 | Buffer + stretch modules as needed · capstone adopted by ≥1 external org · interview → offer |

## Layout

```
platform-eng/
├── platform/<NN>-<slug>/      # Track A — deepen modules (lessons/, practice/, resources/, checkpoint.md)
├── platform/interview-refresh/ #          light refresh pages for the areas you already operate
└── modules/<NN>-<slug>/       # Track B — GPU modules (map to Notion) + Track C in module 12
      └── resources/depth-map.md  #        lesson-level pointers into the external depth library
docs/
├── ROADMAP.md             # month-by-month calendar, milestones, readiness gate
├── CONVENTIONS.md         # structure, mandatory fields, definition of done
├── EXTERNAL-DEPTH.md      # the external depth library — index of all 17 depth maps
└── templates/             # module.md and lesson.md starting points
```

## Two things to know before you start

**You don't need GPUs to start.** The
[**fake GPU fleet lab**](platform-eng/modules/04-gpu-on-kubernetes/practice/fake-gpu-fleet/) stands up a
50–200-node heterogeneous fleet with forged extended resources and a synthetic DCGM exporter, so
Modules 04, 05, 06, 11 and the capstone are all buildable on a laptop. Rent one real GPU for one
afternoon at the end of Module 05 to *validate* the simulation — not to develop against it.

**Depth reading is a lesson input, never a substitute.** Every module carries a
`resources/depth-map.md` pointing into [`harut8/system-design`](https://github.com/harut8/system-design),
a ~400k-line systems-internals library, at lesson granularity. Open a chapter when an artifact is
blocked on internals you don't have — see [`docs/EXTERNAL-DEPTH.md`](docs/EXTERNAL-DEPTH.md) for
the index, the honest limits of that source, and the attribution note. *Build, do not read* still
governs: a lesson is `done` only when its artifact exists.

Both module and lesson files carry **YAML frontmatter** with a `status` field so
progress is machine-readable. See [`docs/CONVENTIONS.md`](docs/CONVENTIONS.md) for
the field spec and the definition of done.

## Workflow per module

1. Work each **lesson** in order: read enough to make the practice possible, then
   build the practice artifact and link it.
2. Answer the **checkpoint** from memory.
3. Flip the module's `status` (frontmatter) and update its tracking row in Notion.

A lesson is `done` only when it has a real artifact; a module is
`checkpoint-passed` only when every lesson is done and the checkpoint is answered.
