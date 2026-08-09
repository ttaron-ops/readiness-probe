# 🎓 Readiness Probe — AI / GPU Infrastructure Study Repo

Working repository for the [**AI / GPU infrastructure study plan**](https://app.notion.com/p/3b23abaeb82380beb8a1d5dd935b9fd0).
Notion holds the plan, the tracking, and the checkpoint syllabus; **this repo holds
the work** — notes, code, benchmarks, and checkpoint answers. Each module in Notion
links to its directory here.

> **Target:** Senior / Staff AI or GPU Infrastructure Engineer.
> **Timeline:** 15–18 months at ~8–10 hrs/week.
> **Operating principle:** *Build, do not read.* The reading exists to make the
> practice possible, and the checkpoints — not the reading — are the real syllabus.

## Layout

```
modules/<NN>-<slug>/
├── README.md       # module overview, links back to the Notion page, status
├── checkpoint.md   # answers to the module checkpoint (the completion gate)
├── notes/          # concept notes
├── practice/       # code, benchmarks, commands — the buildable output
└── resources/      # saved references, diagrams, papers, link index
```

## Modules

Modules run in phases. `01b`/`02b` are parallel/prerequisite sub-modules, not
afterthoughts (`02b` must precede `03`). `12` runs alongside from month 6 onward.

| # | Module | Phase | Directory |
|---|--------|-------|-----------|
| 01  | 🐹 Go for infrastructure engineers | Months 1–5 | [`modules/01-go-for-infra`](modules/01-go-for-infra) |
| 01b | 🐧 Linux systems internals | Months 1–5 (parallel with 01) | [`modules/01b-linux-internals`](modules/01b-linux-internals) |
| 02  | ☸️ Kubernetes internals and controllers | Months 1–5 | [`modules/02-kubernetes-controllers`](modules/02-kubernetes-controllers) |
| 02b | 🖥️ Host hardware and topology | Months 5–8 (precedes 03) | [`modules/02b-host-topology`](modules/02b-host-topology) |
| 03  | 🔌 GPU hardware fundamentals | Months 5–8 | [`modules/03-gpu-hardware`](modules/03-gpu-hardware) |
| 04  | 📦 GPU on Kubernetes | Months 5–8 | [`modules/04-gpu-on-kubernetes`](modules/04-gpu-on-kubernetes) |
| 05  | 📊 GPU observability and telemetry | Months 5–8 | [`modules/05-gpu-observability`](modules/05-gpu-observability) |
| 06  | 📋 Scheduling, queueing and capacity | Months 8–12 | [`modules/06-scheduling-capacity`](modules/06-scheduling-capacity) |
| 07  | 🚀 Inference serving | Months 8–12 | [`modules/07-inference-serving`](modules/07-inference-serving) |
| 08  | 🏋️ Distributed training infrastructure | Months 12–16 | [`modules/08-distributed-training`](modules/08-distributed-training) |
| 09  | 🌐 Networking and topology | Months 12–16 | [`modules/09-networking-topology`](modules/09-networking-topology) |
| 10  | 🔧 Bare metal and cluster lifecycle | Months 12–16 | [`modules/10-bare-metal-lifecycle`](modules/10-bare-metal-lifecycle) |
| 11  | 💰 GPU cost and unit economics | Months 8–12 (ongoing) | [`modules/11-gpu-cost-economics`](modules/11-gpu-cost-economics) |
| 12  | 🎓 Capstone project and interview prep | Months 12–18 | [`modules/12-capstone-interview`](modules/12-capstone-interview) |

## Sequencing

| Phase | Modules | Outcome |
|-------|---------|---------|
| Months 1–5  | 01, 01b, 02 | Go proficiency, Linux depth, a working operator |
| Months 5–8  | 02b, 03, 04, 05 | Host topology, GPU stack hands-on, metrics MVP shipped |
| Months 8–12 | 06, 07, 11 | Queueing, inference, cost attribution working |
| Months 12–16| 08, 09, 10 | Training, networking, bare metal coverage |
| Months 12–18| 12 | Project public and adopted, writing published, interviewing |

**Minimum viable path** (if only half gets done): 01, 01b, 02, 04, 05, 11, plus
Kueue from 06 and vLLM from 07. Modules 08, 09, 10 are the deferrable ones.

## Workflow per module

1. Read enough to make the practice possible (`notes/`, `resources/`).
2. Build the practice artifacts (`practice/`) — this is the point.
3. Answer the checkpoint from memory (`checkpoint.md`).
4. Tick the module's status and update its tracking row in Notion, linking the
   files here as evidence.
