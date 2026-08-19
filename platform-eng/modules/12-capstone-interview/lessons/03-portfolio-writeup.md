---
lesson: 03
title: "Portfolio & writeup build: the repo, the design doc, the diagram"
module: 12
concept: "evidence a manager reads cold"
status: not-started
est_time: "6 hrs"
prev: "02-capstone-synthesis.md"
next: "04-flagship-blog-demo.md"
artifacts: ["a published capstone repo README (whole-fleet architecture diagram + value-chain narrative + chapter links), one 5-8 page RFC-style architecture design doc, and a brag-doc one-pager"]
sources: 11
---

# Portfolio & writeup build: the repo, the design doc, the diagram

[🎓 12 — Capstone & interview preparation](../README.md)

## Where this fits

Lesson 02 gave you the *narrative*: the four-verb thesis (observable, attributable, schedulable, survivable) derived as a dependency chain, the five featured chapters selected by a mechanical filter, the rejected-alternatives bank with reversal conditions, and a rehearsed opener at four compressions. That narrative lives in your head and in a storyboard, and it converts nobody by itself — every one of its delivery surfaces requires you to be in the room. This lesson turns it into **published evidence a hiring manager reads cold**: a repo README, an RFC-style design doc, and a brag-doc, none of which need you present to land. The gap 03 fills is exactly that translation — narrative to artifact — and everything downstream (the flagship post in lesson 04, the design reps in lesson 05, the narration in lesson 07) assumes this published evidence already exists.

## Why this matters

The narrative in your head converts nobody. A repo that opens with one architecture diagram and one value-chain sentence, a design doc that reasons in Goals / Non-goals / Alternatives / Risks, and a one-page quantified brag-doc — those convert, in the four minutes a reviewer spends on your GitHub before deciding whether to route you to a senior loop or a staff one.

The specific trap for a strong IC is *exhaustiveness*. You have eleven artifacts and the instinct is to list all eleven with equal weight. That reads as a portfolio of exercises, not a platform. Lesson 02 already ran the selection filter; this lesson's job is to make sure the published surface honours it — three to five standout pieces carrying the story, at least one at production scale, each with a metric and a post-launch evolution, and the rest demoted to a chapter index. The reviewer is not grading completeness. They are looking for judgement: what you chose to build, what you chose *not* to (the non-goals), and whether you can defend a tradeoff in writing the way you would defend it in a design review.

The GPU-specific reason this matters: your generic platform credibility is assumed at this level, so the writeup has to surface the parts a competent generalist SRE could *not* have written. The time-slicing attribution hole and its mechanism, the util-versus-work gap in field-ID terms, the MIG-versus-time-slice-versus-passthrough tradeoff with a stated verdict, topology-aware placement, the FOCUS columns your schema actually populates. **If your README and design doc could have been produced by someone who has never operated a GPU fleet, they have failed regardless of how polished they are.**

There is also a documented reason this specific artifact shape works: real infrastructure organisations build their design and review culture around exactly this format. Surveys of RFC and design-doc practice across very different companies find the same skeleton recurring — Goals, Non-goals, Alternatives, Risks — not because of fashion but because it is the format a review board actually uses to evaluate judgement rather than prose. Writing your design doc in that voice is not a genre exercise; it produces the same document type a staff-level design review would expect, before anyone asked for one.

## What's new here

- **You already know** how to write clean READMEs, structure a repo, and draw boxes and arrows. **Skip** Markdown mechanics. The new thing is the *editorial* discipline: front-door diagram, value-chain narrative, deliberate demotion of everything else.
- **You already know** how to write a design doc at work. **Skip** learning the RFC skeleton. The new thing is what **staff-level GPU content** looks like inside each section — specifically Alternatives (the partitioning table with a verdict) and Risks (the attribution hole with a *derived* error bound).
- **New**: a **complete worked README and a complete worked design doc**, written out, not described. You edit them into your own rather than starting from a blank file.
- **New**: the **rendering constraints** that decide what actually appears on your published page — no Liquid, no mermaid on the stock theme, relative links needing explicit `README.md` — because a diagram that does not render is worse than no diagram.
- **New**: **FOCUS alignment as a checkable claim**, with the specific columns to name and the two extensions the standard genuinely lacks.
- **New**: the **brag-doc** as a distinct working artifact with a strict one-metric-per-line rule, and what you do and do not publish from it.

## Core concepts

### 1. The reader model, and where they drop off

Everything in this lesson is optimised against one reader: a hiring manager or senior engineer with your tab open, twenty other tabs, and about four minutes. They are not reading. They are running a search for three tokens — *what kind of engineer is this*, *is there a real number*, *can they hold a system in their head* — and they abandon the moment the search fails.

```
   THE COLD READ — where a reviewer drops off, and what stops them
  ══════════════════════════════════════════════════════════════════════════════

   t = 0 s     lands on repo root
      │
      ▼
   ┌────────────────────────────────────────────┐
   │ Is there an IMAGE in the first screen?     │── no ──▶ scroll once, then
   │ (the whole-platform diagram)               │          BOUNCE  ▸▸ ~60% lost
   └───────────────┬────────────────────────────┘
                  yes  (t ≈ 5 s)
                   ▼
   ┌────────────────────────────────────────────┐
   │ Does ONE sentence tell me what this is?    │── no ──▶ reads the file tree,
   │ (the value-chain / thesis line)            │          forms own theory,
   └───────────────┬────────────────────────────┘          usually the wrong one
                  yes  (t ≈ 20 s)
                   ▼
   ┌────────────────────────────────────────────┐
   │ Is there a NUMBER above the fold?          │── no ──▶ classifies as
   │ ("99% util / 0.16 SM_ACTIVE, same GPU")    │          "study project"
   └───────────────┬────────────────────────────┘          ▸▸ level-down, silent
                  yes  (t ≈ 45 s)
                   ▼
   ┌────────────────────────────────────────────┐
   │ Is there ONE obvious next click?           │── no ──▶ opens a random
   │ (flagship post · design doc)               │          subdirectory
   └───────────────┬────────────────────────────┘
                  yes  (t ≈ 60 s)
                   ▼
   ┌────────────────────────────────────────────┐
   │ DESIGN DOC: does it have Non-goals and an  │── no ──▶ "competent, hasn't
   │ Alternatives section WITH A VERDICT?       │          run a design review"
   └───────────────┬────────────────────────────┘
                  yes  (t ≈ 3 min)
                   ▼
   ┌────────────────────────────────────────────┐
   │ RISKS: is the hardest limitation named,    │── no ──▶ "polished, probably
   │ quantified, and DERIVED?                   │          hasn't hit the wall"
   └───────────────┬────────────────────────────┘
                  yes  (t ≈ 4 min)
                   ▼
              ✦ ROUTED TO A STAFF LOOP ✦
```

**Read the failure column, not the success column.** Every drop-off produces a *silent* misclassification, not a rejection you hear about. That is what makes this worth deliberate work: you never get told that your README got you routed one level down.

### 2. Three documents, three readers, one dataset

| Document | Reader | Read in | Must contain | Fails when |
|---|---|---|---|---|
| **Repo README** | hiring manager, recruiter, anyone with a link | 30–60 s, skimming | one diagram, one thesis sentence, one number, one next-click, a chapter index | it is a file-tree explanation, or the diagram is a link rather than inline |
| **Design doc** | senior/staff engineer, design-review reflex engaged | 3–8 min, reading properly | Goals with numbers, Non-goals, Context with a forcing function, Design term by term, Alternatives **with a verdict**, Risks **with a derivation**, Rollout with gates | Alternatives missing or verdict-free; Risks generic; Rollout is "deploy it" |
| **Brag-doc** | **you**, before an interview | 60 s, scanning for a story | one line per outcome, each with a metric and what it unblocked | it becomes a narrative CV, or a list of activities with no numbers |

The brag-doc is not published. It is the working log you mine, so its bar is different: completeness over polish. The README and the design doc are published, so their bar is the cold read.

```
   ONE DATASET, THREE DOCUMENTS — what flows where
  ══════════════════════════════════════════════════════════════════════════════

        LESSON 02 STORYBOARD
        thesis · 5 featured chapters · rejected-alternatives bank · numbers
                       │
        ┌──────────────┼───────────────────────────┐
        ▼              ▼                           ▼
   ┌─────────┐   ┌──────────────┐          ┌───────────────┐
   │ README  │   │ DESIGN DOC   │          │  BRAG-DOC     │
   ├─────────┤   ├──────────────┤          ├───────────────┤
   │ thesis  │◀──│ Goals        │          │ every outcome │
   │ → the   │   │ Non-goals    │          │ one metric    │
   │ value-  │   │ Context      │◀─────────│ per line      │
   │ chain   │   │ Design       │  numbers │               │
   │ line    │   │ Alternatives │◀─────────│ ← mined here  │
   │         │   │  (bank, §5)  │  verdicts│               │
   │ diagram │──▶│ Risks        │          │ NOT PUBLISHED │
   │ number  │──▶│ Rollout      │          │               │
   │ index   │   └──────┬───────┘          └───────┬───────┘
   └────┬────┘          │                          │
        │               │                          └──▶ lesson 07 narration
        │               └──▶ lesson 05 design drills      lesson 08 STAR bank
        └──▶ lesson 04 flagship post links back here
```

### 3. The front-door README, written out

Below is a complete README you can edit into your own. It is not an outline. Everything in it is drawn from this course's actual findings, and every number in it is one you will have measured by the time you publish.

```markdown
# gpu-platform-capstone

A reference GPU platform: making a multi-tenant GPU fleet **observable,
attributable, schedulable and survivable**.

[ ── whole-platform architecture diagram goes here, inline ── ]

**Start here:** [Your GPU dashboard is lying to you](blog/dashboard-is-lying.md)
· [Architecture design doc](design-doc.md)

---

## The one-paragraph version

The metric every GPU dashboard leads with does not measure work. On a batch-1
decode server this fleet reported **99% GPU utilisation while 16% of the SMs
were lit and the tensor pipes ran at 1%** — same GPU, same second. This repo is
the platform built on the honest number instead: an exporter and a Kubernetes
controller that join per-GPU hardware counters to the kubelet's allocation
record, per pod, with a conservation identity that proves the shares on any
physical GPU sum to exactly 1.000. It reports **allocated** and **utilised**
GPU-hours as separate ledgers, decomposes the gap into six buckets with six
different owners, and prices them against a dated rate card.

Most results here come from a **200-node simulated fleet** with the traces
published; one rented GPU-afternoon was used to validate the simulation, not to
develop against it. Where attribution is an estimate — time-sliced devices — the
series says so in a label.

## The value chain

| Layer | The question it answers | Chapter |
|---|---|---|
| **Survivable** | What happens when the hardware fails? | [08 survive-a-failure](chapters/08-survive-a-failure/README.md) · [failure-mode log](chapters/04-attribution/failure-modes.md) |
| **Schedulable** | Who gets the GPU next, and where? | [06 Kueue showback](chapters/06-kueue-showback/README.md) · [02b topology](chapters/02b-topology/README.md) |
| **Attributable** | Whose GPU-hours are these, and what did they cost? | [04 per-pod attribution](chapters/04-attribution/README.md) · [11 cost schema + tooling teardown](chapters/11-focus/README.md) · [07 cost per 1M tokens](chapters/07-unit-economics/README.md) |
| **Observable** | Is the GPU actually working? | [05 dashboard is lying](chapters/05-observability/README.md) · [01 exporter](chapters/01-exporter/README.md) |
| **The machine** | What is it all standing on? | [01b container anatomy](chapters/01b-container/README.md) · [09 fabric read](chapters/09-network/README.md) · [10 bare metal](chapters/10-baremetal/README.md) |

## The four numbers

| Finding | Number | Where it came from |
|---|---|---|
| The util gap | `GPU_UTIL` 99 vs `SM_ACTIVE` 0.16 vs `TENSOR` 0.011, one GPU, one second | [05](chapters/05-observability/README.md) |
| Idle share of allocated GPU-hours | 57.3% over 24 h on a 24-GPU cluster | [05](chapters/05-observability/README.md) |
| Attribution conservation | Σ shares + unallocated ≡ 1.000 per physical GPU, asserted continuously | [04](chapters/04-attribution/README.md) |
| Chargeback resting on an estimate | 46.8% of utilisation-based dollars on a fleet 37.5% time-sliced | [04](chapters/04-attribution/README.md) |

## What this is not

Not a scheduler replacement. Not a multi-cloud cost platform. Not a per-kernel
profiler. Not a product — a reference implementation with its measurements
published so you can disagree with the method rather than the conclusion.
```

Six properties of that README are doing the work, and each maps to a drop-off point in §1:

1. **The diagram is inline and first.** Not a link, not below a badge row.
2. **The thesis is one sentence and it is the subtitle**, so it is read before anything else.
3. **A number appears in the first paragraph** — and it is a *comparison*, not a magnitude, because a comparison is self-explanatory ("99 versus 0.16" needs no context; "0.16 SM_ACTIVE" does).
4. **Start-here appears above the value chain**, because a reader who wants depth should not have to scroll past a table to find it.
5. **The honesty caveat is in the first paragraph, stated as a specification** rather than appended as a confession.
6. **"What this is not" is a Non-goals section on the front door.** Almost nobody does this, and it is one of the cheapest staff signals available, because it proves scope was a decision.

### 4. The whole-platform diagram, and the rendering constraints

Draw the fleet as data flowing, one layer per row, arrows labelled with *what flows* rather than merely connecting boxes. Mark two things explicitly, because they are the two places your platform differs from a generic one: **where the attribution join happens**, and **where the honest metric replaces the duty cycle.**

```
   REFERENCE GPU PLATFORM — the front-door diagram
  ══════════════════════════════════════════════════════════════════════════════

   ┌──────────────┐   ┌──────────────┐   ┌──────────────┐   ┌──────────────┐
   │  GPU NODE 1  │   │  GPU NODE 2  │   │     ...      │   │  GPU NODE N  │
   │  8 × H100    │   │  8 × H100    │   │              │   │  8 × A100    │
   └──────┬───────┘   └──────┬───────┘   └──────┬───────┘   └──────┬───────┘
          │ HW counters      │                  │                  │
          ▼                  ▼                  ▼                  ▼
   ┌─────────────────────────────────────────────────────────────────────┐
   │ PER-NODE AGENTS  (DaemonSet)                                        │
   │                                                                     │
   │   NVIDIA device plugin ──── allocates ────▶ kubelet pod-resources   │
   │        │                                        │  ListPodResources │
   │        │                                        │  {ns, pod, [UUID]}│
   │   nv-hostengine / DCGM ── field cache ──┐       │                   │
   │        │  1002 SM_ACTIVE  ◀── THE HONEST METRIC │                   │
   │        │  1004 TENSOR_ACTIVE             │      │                   │
   │        │  203  GPU_UTIL  ◀── kept ONLY as the foil                  │
   │        ▼                                 ▼      ▼                   │
   │   ╔═══════════════════════════════════════════════════════════════╗ │
   │   ║  ★ THE ATTRIBUTION JOIN ★   allocation ╳ measurement          ║ │
   │   ║      meet at exactly one key: the GPU UUID                    ║ │
   │   ║      invariant:  Σ allocated_share + unallocated ≡ 1.000      ║ │
   │   ╚════════════════════════════┬══════════════════════════════════╝ │
   └────────────────────────────────┼────────────────────────────────────┘
                                    │ :9401/metrics
                    per-pod shares, labelled with:
                    attribution ∈ {exact, per-pid, shared-estimate}
                    share_basis ∈ {exclusive, mig-compute, mig-memory, replica}
                                    ▼
   ┌─────────────────────────────────────────────────────────────────────┐
   │ PROMETHEUS ── recording rules ──▶ Grafana                           │
   │      │                             · allocated vs utilised          │
   │      │                             · the six-bucket gap             │
   │      │                             · the util-lie detector panel    │
   │      ├──▶ RECONCILIATION ALERTS  (identity A ≠ 1.000 → page)        │
   │      │                                                              │
   │      ├──▶ KUEUE  quota · borrow/reclaim · gang  ──▶ scheduling      │
   │      │                                                              │
   │      └──▶ COST PIPELINE ──▶ FOCUS 1.4 rows + 2 x_ extensions        │
   │                              (x_UtilisedEffectiveCost,              │
   │                               x_SplitConfidence)                    │
   └─────────────────────────────────────────────────────────────────────┘

   RATE CARD is data, not code: node labels →  $/physical-GPU-hour,
   with the basis and the date carried alongside the number.
```

**Rendering constraints that decide whether any of this survives publication.** These are not style notes; each one is a way your page silently breaks.

- **Liquid templating is disabled site-wide.** Do not use `{%` tags of any kind, and remove any that exist in files you inherit. Braces inside code blocks are fine as long as no Liquid tag is formed.
- **The stock GitHub Pages theme does not render mermaid.** A mermaid block renders as a wall of source text on the published site while looking fine in the GitHub file view — which is the worst possible failure, because you will not notice. **ASCII inside a fenced code block renders identically in both places**, which is why every diagram in this course is ASCII.
- **Relative links to a directory need the explicit `README.md`.** `[chapter](chapters/04-attribution/)` resolves on github.com and 404s on the built site. Always `chapters/04-attribution/README.md`.
- **One diagram, one screen, no horizontal scroll.** Keep lines under about 78 characters or a phone viewport will wrap them into noise.

**Test it the only way that works:** build the site, open it on a phone, and read the front page. Everything this section warns about is invisible until you do.

### 5. The design doc, written out

Five to eight pages, real design-doc voice: declarative, tradeoff-forward, addressed to a review board. Here is the doc, with each section written rather than described. Edit the numbers to your own.

---

> #### RFD 001 — Per-pod GPU cost attribution for a multi-tenant fleet
>
> *Status: accepted · Authors: … · Created 2026-08-01 · Last revised 2026-08-14*
>
> **Goals**
>
> 1. Attribute **≥95% of GPU-hours** to a namespace within one billing day, on a fleet
>    containing exclusive, MIG and time-sliced devices.
> 2. Publish **allocated** and **utilised** GPU-hours as two separate series, never one blended
>    number, so that a scheduling claim and an efficiency claim can be told apart.
> 3. Hold the conservation identity **Σ allocated_share + unallocated ≡ 1.000 per physical GPU**
>    continuously, asserted as an alerting rule rather than a one-off check.
> 4. Keep agent overhead **below 1% of one GPU per node** and Prometheus cardinality growth
>    **below 40 series per GPU**.
> 5. Emit cost rows conformant to **FOCUS 1.4**, with every non-standard field carried as an
>    `x_` extension rather than a bespoke column name.
>
> **Non-goals**
>
> - **Not a scheduler.** This system informs allocation decisions; it does not make them. Kueue
>   owns admission and quota.
> - **Not hard chargeback in v1.** Showback only. Chargeback requires a dispute protocol and a
>   validated ledger; both are v2.
> - **Not per-kernel profiling.** The escalation ladder ends at DCGM field level; Nsight and
>   PyTorch Profiler are a tenant-side activity, deliberately out of scope.
> - **Not multi-cloud.** One provider's rate card in v1. The schema is FOCUS-shaped precisely so
>   this stays a data problem later rather than a redesign.
> - **Not an exact per-tenant split under time-slicing.** See Risks R1 — this is a physical
>   limit, and pretending otherwise would make every other number in the system untrustworthy.
>
> **Context**
>
> The fleet is 24 GPUs across 3 nodes (16 × H100 80GB, 8 × A100 80GB) in the reference
> deployment, and the design targets 200+ nodes. At **$2.50/GPU-hour for H100 and $1.60 for
> A100 — 1-year-reserved basis, February 2026 rate card, a dated snapshot** — the reference
> fleet costs $1,267/day, about $463k/year.
>
> The forcing function is that **none of that spend is currently attributable, and the metric
> everyone is using to reason about it does not measure work.** `DCGM_FI_DEV_GPU_UTIL` (field
> 203) is an unmodified passthrough of NVML's `nvmlUtilization_t.gpu`, defined in the header as
> the percentage of a driver-chosen sample window during which *at least one kernel was
> resident.* It is a threshold at one, with no notion of how many SMs exist. Measured on this
> fleet: a serving namespace reads 99–100 on field 203 while `DCGM_FI_PROF_SM_ACTIVE` reads
> 0.16 and `PIPE_TENSOR_ACTIVE` reads 0.011 — 86% of the compute die dark behind a green
> dashboard.
>
> Integrating the honest metric over 24 hours gives **231.4 utilised GPU-hours against 576
> allocated**: 57.3% of the invoice buying no SM work. The largest single contributor is a
> serving namespace holding 8 GPUs at 11.0% mean SM-active — which, on the dashboard everyone
> actually looks at, is the busiest namespace in the cluster.
>
> **Design**
>
> Two loops, one join, two ledgers.
>
> *Ownership loop (10 s).* Call the kubelet's pod-resources `List()` over its unix socket,
> resolve every returned device ID into the string that will appear on the DCGM series, and
> publish an immutable snapshot. Four device-ID shapes must be resolved: `GPU-<uuid>` (whole
> device, 1:1), `MIG-<uuid>` (requires an NVML hop — `DeviceGetHandleByUUID` →
> `GetDeviceHandleFromMigDeviceHandle` → `GetGpuInstanceId` — because dcgm-exporter labels a MIG
> sample with the *parent* UUID plus `GPU_I_ID`, not the MIG UUID), `GPU-<uuid>::<replica>`
> (time-sliced or MPS; N holders, one measurement identity), and DRA claims (resolved through
> the allocated device's CDI name).
>
> The owner map is typed `map[string][]Owner`, not `map[string]Owner`. That is the entire
> time-slicing correction encoded in the type: a join key can have more than one holder, and a
> single-valued map silently keeps whichever holder was written last — which charges one pod for
> the whole card and the others for nothing.
>
> *Utilisation loop (15 s).* Read DCGM per device and per GPU instance; where a device is
> shared, additionally attempt NVML per-PID data and map PID → pod UID through
> `/proc/<pid>/cgroup`. Record which rung of the fidelity ladder actually worked, as a label.
>
> *The join.* The two loops meet at exactly one key: the physical GPU UUID. Everything
> downstream depends on that one join being correct, which is why it is the thing the
> reconciliation identities test.
>
> *The share model.* `allocated_share` is a pod's claim on one physical GPU in [0,1], computed
> without reading any metric: 1.0 for exclusive; `inst.SMCount / full.SMCount` for MIG on a
> compute basis (3g.40gb on an A100-80GB = 42/98 = 0.4286) or `inst.MemMiB / full.MemMiB` on a
> memory basis (0.5000 for the same slice — the bases disagree by up to about 16% relative and
> **the chosen basis is emitted as a label, never averaged**); and `1 / replicas` for shared
> devices — *replicas configured*, not holders present, so an over-provisioned replica count
> stays a platform cost rather than being socialised onto tenants.
>
> *Two gauges, never collapsed.* `gpu_allocated_share` (a scheduler fact) and
> `gpu_utilised_share` (a hardware measurement) have different denominators and different
> trustworthiness. Cost is emitted from both: `gpu_cost_allocated_per_hour` is what you bill,
> `gpu_cost_utilised_per_hour` is the efficiency signal.
>
> *Cardinality budget.* Ten labels × ~4 series per GPU ≈ 40 series/GPU. At 200 nodes × 8 GPUs
> that is ~64k series, which is a Prometheus sizing input, not a rounding error, and it is why
> per-container labels are capped and per-PID series are not exported.
>
> **Alternatives considered**
>
> *(A) Partitioning model.*
>
> | Option | Isolation | Attribution | Utilisation | Stranding |
> |---|---|---|---|---|
> | **Passthrough** | total — one device, one pod | trivially exact | poor if the tenant cannot fill a card | none |
> | **MIG** | hard — separate SMs, memory paths, cache | exact; an instance belongs to one container | rigid; partitions do not rebalance under shifting traffic | real: 7 × `1g.10gb` sums to 0.858 of the card on a memory basis — 14.2% physically unallocatable |
> | **Time-slicing** | none — context-switched, shared memory | **estimate only**; one device counter, N tenants | high for bursty tenants | none, but no isolation |
> | **MPS** | partial — thread-percentage cap | bounded estimate, because the cap bounds the error | high | none |
>
> **Chosen:** MIG for the shared-tenant tier, passthrough for large single-tenant training,
> time-slicing confined to the dev tier. The deciding constraint is Goal 1 — ≥95% of GPU-hours
> attributable — and only MIG and passthrough produce an exact split. Time-slicing survives in
> dev because isolation and per-tenant dollars genuinely do not matter there, and it is the only
> option that gives interactive notebooks acceptable density.
>
> **Reversal condition:** if the shared tier's traffic mix becomes bursty enough that MIG's
> static geometry strands more capacity than time-slicing wastes, revisit. Measure it as
> stranded GPU-hours per week versus the dev tier's unattributable hours.
>
> *(B) Utilisation integration.* `avg_over_time(x[24h]) * 24` versus
> `sum_over_time(x[24h]) * Δ/3600`. **Chosen:** `sum_over_time`. `avg_over_time` averages only
> over samples that exist, so a workload running 9 hours of a 24-hour window has its mean
> extrapolated across the whole window — an overstatement of exactly `window ÷ time-present`,
> which is 2.67× in that case. It errs in the direction that makes the fleet look more utilised,
> i.e. it hides the problem on precisely the bursty workloads where the problem is largest.
>
> *(C) Build versus extend the incumbent.* The leading open-source Kubernetes cost engine bills
> GPUs as `GPUCost = GPUHours × CostPerGPUHr`, where `GPUHours` comes from a whole-device
> resource *request* count. That expression has exactly two factors — an integer count and a
> price per physical device — and a sub-device fraction is neither. **Chosen:** build the
> correction layer, and upstream the two gaps that genuinely are small (adding the shared
> resource key to both numerator paths; repairing an inverted guard on a vGPU scaling branch).
> The fractional numerator is an input-model change and needs a decision on the fractional basis
> that the standard deliberately declines to prescribe.
>
> **Risks**
>
> **R1 — Per-tenant attribution under time-slicing is an estimate, and cannot be made exact.**
> The mechanism is hardware: the profiling counter is scoped to the *device*, not to the CUDA
> context that happened to be resident when it was sampled, so N pods sharing one GPU all read
> the same aggregate value. No code in any cost engine recovers N per-tenant values from one
> device value; the information was never produced.
>
> *Quantified, with the derivation.* For a device at utilisation `u` shared by `N` pods with
> true shares `sᵢ` (Σ sᵢ = 1), a fair-share estimator assigns `u/N`, so the error for pod *i* is
> `errᵢ = u × (1/N − sᵢ)`. Two properties follow: **Σ errᵢ = u × (1 − 1) = 0**, so the error is
> purely redistributive and the *device total is exactly right* — every dollar is accounted for,
> some on the wrong invoice; and `|errᵢ| ≤ u`, bounded by that device's own utilised cost.
>
> Aggregating to the fleet: with 40 of 64 GPUs exclusive or MIG at mean utilisation 0.62, and 24
> time-sliced at mean utilisation 0.91, exact utilised GPU-equivalents are 40 × 0.62 = 24.80 and
> estimated ones are 24 × 0.91 = 21.84, so the **exposure fraction is 21.84 / 46.64 = 46.8%**.
> At $4.00/GPU-hour × 730 h that is **$63.8k/month of chargeback resting on an estimate.**
>
> *Mitigation.* (i) Every series carries an `attribution` label — `exact`, `per-pid`,
> `shared-capped-estimate`, `shared-estimate` — so the exposure fraction is a query, not a
> guess. (ii) Bill from `gpu_cost_allocated_per_hour`, which has no such exposure because
> allocation is a scheduler fact rather than a measurement. (iii) Getting NVML per-PID
> attribution working on the shared tier moves those dollars from `shared-estimate` to
> `per-pid` and drops the exposure to near zero — which is the roadmap item, stated in dollars
> rather than in engineering preference.
>
> **R2 — The honest metric is opt-in and its absence is silent.** `DCGM_FI_PROF_SM_ACTIVE`
> ships commented out in dcgm-exporter's default counter CSV, and requires elevated privileges
> the device-level fields do not. When it is unavailable the exporter emits *nothing* rather
> than a zero, and absence in PromQL is not zero — `avg by (namespace)` silently drops the GPU,
> so a fleet where half the cards stopped reporting shows an unchanged, healthy-looking average.
> *Mitigation:* an alerting rule on absence (`count(device field) unless count(PROF field)`),
> plus the standing rule **never emit a zero you did not measure.**
>
> **R3 — Label-collision misattribution.** The exporter emits `namespace`/`pod`/`container`;
> Prometheus's Kubernetes service discovery attaches target labels of the same names describing
> *the exporter's own pod*. With `honor_labels: false` (the default) the scraped labels are
> renamed with an `exported_` prefix, so grouping by `namespace` attributes the entire cluster's
> GPU-hours to the exporter's namespace — a dashboard that looks completely plausible and is
> entirely wrong. *Mitigation:* one recording rule normalises to a single `ns` label with an
> explicit `__unallocated__` fallback, and every downstream query is written against that rule
> rather than the raw series.
>
> **R4 — Rate-card drift.** The dollar layer is the most challengeable part of the system.
> *Mitigation:* rates are data (a metric with `modelName` labels, versioned in git), never
> constants in code; every published figure states its basis and its date; and the unknown-model
> fallback is a deliberate, visible value rather than a silent zero.
>
> **Rollout**
>
> | Phase | Scope | Gate to advance | Rollback trigger |
> |---|---|---|---|
> | 0 · shadow | agents deployed, metrics emitted, no dashboard published | identity A holds at 1.000 ± 0.001 for 7 consecutive days; sample completeness ≥98% | any identity-A violation not explained within a day |
> | 1 · one namespace | dashboard shared with one tenant team, showback only | synthetic ground-truth test within ±2 scrape intervals; tenant agrees the numbers match their own job records | tenant disputes a number and we cannot reproduce it from `x_SourceSnapshotId` |
> | 2 · fleet showback | all namespaces, platform buckets published alongside tenant buckets | attribution coverage ≥95%; exposure fraction published and trending down | coverage regression below 90% |
> | 3 · chargeback | v2, requires the dispute protocol | out of scope for this document | — |
>
> **Validation.** Five checks, all of which must pass before any figure is published: the
> reconciliation identity (six buckets sum to the physical ceiling within ~1%); the ordering
> invariant (utilised never exceeds allocated — any result is a bug); sample completeness (≥98%
> of expected samples, because gaps make waste look smaller); a **synthetic ground-truth run**
> (saturate one GPU for exactly 600 s, expect 0.1667 GPU-hours, accept 0.150–0.184 — 0.30 means
> double counting, ~0.08 means the integration constant is 15 s not 30 s, ~4.0 means the
> `avg_over_time` bug); and external agreement against the invoice, where the invoice must be
> ≥ allocated hours with the difference equal to the cordoned plus unallocated buckets.

---

**What makes that doc read as staff rather than senior**, concretely, so you can check your own:
Non-goals exist and each names something a reasonable reader would otherwise assume; Context
contains a forcing function with a number rather than a description of the environment;
Alternatives states a verdict *and a reversal condition* for each choice; Risks names the hardest
limitation first, derives its magnitude, and separates the parts that are engineering from the part
that is physics; and Rollout has gates and rollback triggers rather than phases. A reviewer who
reads only Alternatives and Risks should be able to reconstruct your entire strategy. That
compression is what the phrase means in practice.

### 6. FOCUS alignment as a checkable claim

If your doc or README says the cost pipeline is "FOCUS-aligned," that is a claim against a real
specification and a FinOps-literate reader will check it. Make it checkable by naming columns.

The relevant state of the standard, which you should state with its version because it has moved:
**FOCUS 1.4 is the current ratified release (announced June 2026)**; the split-cost-allocation
columns you need arrived in **1.3 (December 2025)** — `AllocatedResourceId`, `AllocatedMethodId`,
and `AllocatedMethodDetails`, whose `Elements` array carries per-entry `AllocatedRatio` values that
**must sum to 1 across all allocated charges derived from one origin charge**. The specification
repository's `main` branch carries a 1.5 candidate adding `AllocatedServiceName`.

**The claim you are entitled to make**, and the two extensions that make it honest:

| What you populate | Why |
|---|---|
| `BilledCost`, `EffectiveCost`, `ListCost`, `ContractedCost` | the four cost columns; you allocate on `EffectiveCost` because it is the accrual view and spreads prepaid commitments across the hours they cover |
| `ChargeCategory` = `Usage`, `ChargePeriodStart/End`, `ResourceId`, `ServiceName` | the mandatory dimensional spine |
| `ConsumedQuantity` / `ConsumedUnit` in GPU-Hours | the allocated quantity |
| `AllocatedResourceId`, `AllocatedMethodId`, `AllocatedMethodDetails` | the 1.3 split-cost primitive — this is what lets a MIG instance or a time-slice share be expressed in-standard rather than smuggled into `Tags` |
| **`x_UtilisedEffectiveCost`, `x_UtilisationMean`, `x_UtilisationScope`** | **gap 1** — FOCUS has no utilisation ledger anywhere. Every cost and quantity column in the standard, including the new `Allocated*` ones, is an allocated or billed number; `ConsumedQuantity` is provider-metered, not a measurement of device work |
| **`x_SplitConfidence`, `x_SplitUncertainty`** | **gap 2** — `AllocatedMethodId` records *which* method produced a split and `AllocatedMethodDetails` can carry the ratio, but nothing carries an error band. For MIG, `exact` is the honest value; for time-slicing it is not, and a downstream reader has no in-spec way to learn that |

**The sentence that shows you read the spec rather than a summary:** *"FOCUS 1.4, using the
split-cost-allocation columns that landed in 1.3, plus exactly two `x_` extensions — because the
standard still has no utilisation ledger and no confidence band on an estimated split."* And the
correction worth carrying: the widely repeated claim that FOCUS cannot represent shared or split
cost was true through 1.2 and has been **false since 1.3**. Repeating it in 2026 signals stale
knowledge in the exact area you are claiming as a specialism.

**Also name what you do not populate**, as a Non-goal. `x_UtilisationScope` deserves a special
mention because it is the field most schemas miss: without it, a reader cannot tell whether
`x_UtilisationMean = 0.62` means "this tenant's MIG instance ran at 62%" or "the device this
tenant shares ran at 62% and we have no idea what their share of it was." Those are different
claims and the column that distinguishes them costs nothing.

### 7. The brag-doc

A one-page running log of outcomes, distinct from the resume and from the README. The format is
one line per outcome, and the rule is **one metric per line, or the line does not go in.**

```
   BRAG-DOC LINE SHAPE   ── shipped X → moved metric Y from A to B → unblocked Z
  ══════════════════════════════════════════════════════════════════════════════

   ✓ Built the per-pod GPU attribution exporter (Go, DaemonSet, two loops)
       → conservation identity Σ share + unallocated ≡ 1.000 asserted
         continuously; 0 violations over 30 days on the reference fleet
       → unblocked showback for 3 namespaces without a manual reconciliation

   ✓ Proved the util-lie on our own hardware
       → GPU_UTIL 99 vs SM_ACTIVE 0.16 vs TENSOR 0.011, one GPU, one second
       → killed an 8-GPU expansion request; ~$175–210k/yr avoided at
         $2.50–3.00/GPU-hr (basis and date stated)

   ✓ Corrected the GPU-hours integration across the whole query pack
       → sum_over_time replaces avg_over_time; the old form overstated by
         window ÷ time-present (2.67× on the bursty namespace)
       → validated against a synthetic 600 s burn: 0.1667 expected,
         measured inside the ±2-scrape-interval band

   ✓ Read the incumbent cost tool's source and quantified where it breaks
       → same H100, same price: $2.10 / $14.70 / $0.00 / $8.40 across four
         sharing regimes, each traceable to a named code path
       → turned "why not just use X" from a debate into a two-minute answer

   ✗ (what NOT to write)
       "Improved GPU observability."          ← no metric
       "Worked on cost attribution."          ← activity, not outcome
       "Reduced costs significantly."         ← magnitude with no number
```

You do not publish the brag-doc. You publish its *distilled* numbers in the README's "four
numbers" table, and you mine its lines for lesson 07's narration and lesson 08's STAR bank. Keeping
it current is the cheapest possible insurance against ever scrambling for "a time you had impact"
— and the discipline of refusing metric-free lines is what forces you to go and measure things
while it is still easy.

### 8. Publishing hygiene

A short checklist, because each item has bitten a real published portfolio:

- **Every relative link resolves on the built site**, which means directory links carry the
  explicit `README.md`. Verify by building and clicking, not by reading.
- **No Liquid tags anywhere**, including inside code fences, since Liquid is disabled site-wide.
- **No mermaid**, since the stock theme does not render it; ASCII in fenced blocks renders
  identically in the GitHub file view and on the built page.
- **Every number carries a basis and a date.** `$2.50/GPU-hour` alone is your most attackable
  sentence; `$2.50/GPU-hour, 1-year-reserved basis, February 2026 rate card` is not.
- **Every simulated result says so, first.** "On a 200-node simulated fleet, traces published, I
  measured…" — as a specification, not a confession.
- **Scrub employer specifics.** Rates you were quoted, cluster names, internal hostnames, tenant
  team names. Publishable-by-default means the review happens before the push, not after.
- **The design doc is dated and revisable.** Numbering and dating documents so a reader can see
  them evolve is a habit worth lifting from organisations that publish their design corpus in the
  open — it signals that the document is a living artifact rather than a one-off writing sample.

## Perspectives

**The hiring manager's four-minute skim.** They do not read your repo; they scan it against a fast
unconscious checklist. Does the top tell me what this is in one image and one sentence? Is there a
number above the fold? Does the design doc have Alternatives, or did they only tell me what they
built? Optimise everything here against that reader, not against a patient colleague — in the real
funnel, the patient colleague only appears after the skim already said yes.

**The design-review board.** A staff panel reading your doc is pattern-matching against every
review they have sat in, and they know exactly what a document looks like when the author has not
actually weighed alternatives: no Non-goals, an Alternatives section that is missing or is a list
with no verdict, and a Risks section that says "scaling might be hard." Their real question is
"would I trust this person to run a review for my team," and the tell is whether the easy-to-skip
sections got real attention.

**The FinOps reader.** Someone reading your economics chapter may not be an engineer at all. To
them, R1 is not a technical footnote — it is the difference between a number they can put in a
board deck and one they cannot. Stating the error bound *and its derivation* is exactly what lets a
non-engineer rely on your system instead of quietly discounting it. It is also why the mitigation
section says "bill from the allocated ledger": that sentence tells a finance partner which of your
two numbers is safe to build on.

**The portfolio-as-product view.** Treat the README as a shipped artifact with a front door, not as
documentation of one. A product's first five seconds decide whether the visitor stays; your
diagram-plus-thesis-sentence *is* those five seconds. Product thinking — what is the one action I
want next (open the flagship post), what is the one thing they must understand first (the value
chain) — is the difference between a repo that reads as a code dump and one that reads as a
platform someone deliberately designed, including its presentation.

**The future-you view.** Every one of these documents is also a note to yourself six months from
now, when the details have faded and a recruiter asks about a number. The design doc's Risks
section is the fastest way to reload the whole argument before a loop, and the brag-doc is the only
place the raw outcomes survive. Write them for the version of you who has forgotten.

## Real-world use cases

- **Public engineering design-doc corpora.** Some infrastructure companies publish their entire
  design-discussion corpus openly, numbered and dated, readable cold by anyone. **What it shows:**
  "publish the design doc" is not an invented exercise — it is an existing practice, and a public
  corpus is the closest thing available to an answer key for what a shareable, staff-caliber design
  doc looks like when a stranger opens it. Two structural habits worth lifting: number and date
  documents so their evolution is visible, and write for a reader outside the immediate team — the
  same audience assumption your capstone has to make.

- **Surveys of RFC and design-doc culture across companies.** Very different organisations converge
  on the same Goals / Non-goals / Alternatives / Risks skeleton. **What it shows:** the template
  here is not stylistic preference; it is the format review boards use to evaluate judgement, which
  is why omitting Alternatives is read as a judgement signal rather than as a formatting choice.

- **NVIDIA `dcgm-exporter`'s `etc/default-counters.csv`.** `DCGM_FI_DEV_GPU_UTIL` ships enabled;
  `DCGM_FI_PROF_SM_ACTIVE` and `SM_OCCUPANCY` ship commented out. **What it shows:** the mechanism
  behind design-doc risk R2, in one file. It also demonstrates the citation habit that makes a
  design doc credible — a risk with a file behind it is a different class of claim from a risk with
  an adjective behind it.

- **Prometheus `honor_labels` in `scrape_config`.** When a scraped label collides with a target
  label and `honor_labels` is false — the default — the scraped label is renamed with an
  `exported_` prefix. **What it shows:** the mechanism behind R3, and why a GPU cost dashboard can
  attribute an entire cluster's GPU-hours to the exporter's own namespace while looking completely
  reasonable. This is the highest-value fifteen minutes of reading in the whole build.

- **The FinOps Foundation's FOCUS specification.** The vendor-neutral schema your "FOCUS-aligned"
  claim points at, including the 1.3 split-cost-allocation columns and the `x_` custom-column
  convention. **What it shows:** both a primary source to cite and a checklist for which columns
  you populate — and, in the two gaps it does not cover, the precise justification for your
  extensions.

## Worked example

**A reviewer opens the repo cold.** Here is what happens, minute by minute, against §1's drop-off
model.

**0:00–0:05.** The ASCII platform diagram is the first thing after the title, inline, one screen,
no horizontal scroll. They see five layers with labelled arrows and a double-boxed join point in
the middle. **Drop-off avoided:** there is an image, and it is a system rather than a file tree.

**0:05–0:20.** The subtitle sentence: *"making a multi-tenant GPU fleet observable, attributable,
schedulable and survivable."* They now have a category for you. **Drop-off avoided:** one sentence
told them what this is.

**0:20–0:45.** The one-paragraph version, whose second sentence is *"99% GPU utilisation while 16%
of the SMs were lit and the tensor pipes ran at 1% — same GPU, same second."* That is a comparison,
so it needs no context to be alarming, and it is specific enough to be checkable. Two sentences
later: *"most results come from a 200-node simulated fleet with the traces published."*
**Drop-off avoided:** a number above the fold, and the honesty caveat arrived before they had to
wonder.

**0:45–1:00.** "Start here" gives two links. They click the design doc, because they are an
engineer. **Drop-off avoided:** one obvious next click.

**1:00–2:00.** Goals are numbered and measurable (≥95% attribution, two separate ledgers, identity
holding continuously, <1% overhead, FOCUS 1.4 conformance). Non-goals cut four things a reader
would otherwise assume, including hard chargeback. Context has a forcing function with a dollar
figure and a dated basis. **The reviewer's internal note is forming: this person scopes.**

**2:00–3:00.** Alternatives has the four-row partitioning table and, crucially, closes with
*"Chosen: MIG for the shared-tenant tier… the deciding constraint is Goal 1"* plus a reversal
condition with a measurable trigger. **Note updates: this person decides, and knows when they would
change their mind.**

**3:00–4:00.** Risks opens with R1 — the time-slicing hole — states the mechanism in one sentence
(the counter is scoped to the device, not the context), derives the error algebraically
(`errᵢ = u × (1/N − sᵢ)`, so Σ errᵢ = 0 and the device total is exactly right), aggregates it to a
fleet-level exposure fraction of 46.8% and $63.8k/month, and then gives three mitigations one of
which is "bill from the ledger that has no exposure." **Note finalises: this person has hit the
wall, knows which side of it is physics, and told me before I asked.**

**The failure mode, for contrast.** Eleven equally weighted repo links; no diagram, or a diagram
that is a link to a `.png`; a design doc whose Rollout says "deploy it," whose Alternatives section
is absent, and whose Risks section says "the error is about ±15%" with no derivation. Every one of
those is indistinguishable from a competent SRE who has never touched a GPU — and the last one is
worse than saying nothing, because a specific-looking number with no source reads as invented, and
a reviewer who suspects one invented number stops trusting all of them.

**The self-scoring rubric.** Open your own repo in a private window and time yourself against §1's
seven gates. Any gate you fail is a specific edit, not a vague improvement.

## Practice

Feeds [GPU platform capstone](../practice/gpu-platform-capstone/README.md).

1. **Write the front-door README** using §3 as the starting text. Replace every number with one you
   measured. Keep the "What this is not" section — it is the cheapest staff signal on the page.

2. **Draw the whole-platform diagram in ASCII**, in a fenced code block, under 78 columns wide.
   Label every arrow with what flows. Mark the attribution join and the point where the honest
   metric replaces the duty cycle. **Then build the site and open it on a phone.**

3. **Write the design doc**, all seven sections. Fully write Alternatives (partitioning table +
   verdict + reversal condition) and Risks (R1 with the algebra and the exposure fraction computed
   from *your* fleet's shared-device count and utilisation). Stub the rest and fill it in as later
   lessons produce material — but never ship with Alternatives or Risks stubbed.

4. **Compute your own exposure fraction.** Take your fleet's `G` GPUs, the `G_s` that run a
   software sharing mode, and the mean utilisation of each group; produce
   `(G_s × ū_s) / (G × ū_all)` and the dollar figure at your rate. This single number is the most
   defensible sentence in your whole cost argument, because it quantifies your own uncertainty.

5. **If you claim FOCUS alignment**, write the column list: what you populate from the core schema,
   which 1.3 split-cost columns you use, and the two `x_` extensions with the gap each closes. Add
   the columns you deliberately do not populate as a Non-goal.

6. **Start the brag-doc** with one line per outcome across your top five artifacts, in the shape
   from §7. Delete every line without a metric — do not soften them, delete them, and put the
   deleted ones on a "go measure this" list.

7. **Run the publishing-hygiene checklist** from §8 over everything before pushing: link
   resolution on the built site, no Liquid tags, no mermaid, basis-and-date on every figure,
   simulation labelled first, employer specifics scrubbed.

8. **Ask one person who does not know the project to read the README for four minutes**, then ask
   them three questions: what does this do, what is the strongest number, and what would you click
   next. Every wrong answer is a specific edit.

**Acceptance:** a README passing all seven gates in §1 · one inline ASCII platform diagram that
renders on the built site and on a phone · a design doc with Non-goals, an Alternatives verdict
with a reversal condition, and a Risks section whose headline risk is derived rather than asserted
· your own computed exposure fraction with its dollar figure · a FOCUS column list if you make the
claim · a brag-doc with no metric-free lines · a clean hygiene checklist.

## Common pitfalls

1. **An Alternatives table with no verdict.** **Mechanism:** a table of options with no chosen one
   is a literature review; it demonstrates awareness, not judgement, and a review board reads the
   absence of a verdict as an absence of a decision. **Symptom:** you cannot answer "so which did
   you pick?" without adding information that is not in the doc. **Fix:** one closing sentence
   naming the choice, tying it to a specific Goal or Context constraint, plus the reversal
   condition.

2. **Omitting Non-goals because nothing was "explicitly cut."** **Mechanism:** every real v1 has
   scope cuts, so an absent Non-goals section does not read as "nothing was cut" — it reads as
   "scope was never considered," which is worse than admitting a cut. **Symptom:** a reader assumes
   your system does something it does not, and finds out in the follow-up.

3. **Using mermaid for the diagram.** **Mechanism:** the stock Pages theme does not render it, so
   the block appears as raw source on the published site while looking correct in the GitHub file
   view — you will not notice. **Symptom:** the site shows `graph TD` and a wall of arrows as
   plain text. **Fix:** ASCII in a fenced block, verified on the built site.

4. **A precise-looking error bound with no derivation.** **Mechanism:** an unsourced specific number
   reads as invented, and a reviewer who suspects one invented number discounts every other number
   in the document. **Symptom:** "the attribution error is about ±15%" with nothing behind it.
   **Fix:** show the estimator, the algebra, the aggregation, and the assumptions — `errᵢ = u ×
   (1/N − sᵢ)` takes three lines and converts the whole risk section from assertion into analysis.

5. **Claiming FOCUS alignment without naming columns.** **Mechanism:** it is a claim against a real
   specification, so a FinOps-literate reader treats an unqualified claim as marketing.
   **Symptom:** "our pipeline is FOCUS-aligned" with no column list anywhere in the repo. **Fix:**
   name the core columns, the 1.3 split-cost columns, and the `x_` extensions with the gap each
   closes.

6. **Repeating the stale FOCUS claim.** **Mechanism:** "FOCUS cannot represent shared or split
   cost" was true through 1.2 and false since 1.3 (December 2025). **Symptom:** you assert a gap
   that the standard closed, in your own claimed specialism, to someone who tracks the spec.
   **Fix:** the narrower, stronger claim — FOCUS now has a first-class split primitive; what it
   still lacks is a utilisation ledger and a confidence band on an estimated split.

7. **A brag-doc that drifts into a narrative CV.** **Mechanism:** without the one-metric-per-line
   rule it fills with activities, and an activity log cannot be mined for stories because stories
   need outcomes. **Symptom:** you read it before an interview and find nothing you can say.

8. **Directory links without `README.md`.** **Mechanism:** they resolve on github.com and 404 on
   the built Pages site. **Symptom:** your chapter index works when you test it and is broken for
   everyone who follows the published link.

## Self-check

- **Your capstone has eleven artifacts. Why is listing all eleven as equal-weight entries weaker
  than featuring four or five?** *Answer:* because the reviewer's read is a four-minute search, not
  a study, and eleven equal entries push the synthesis work onto them — which they will not do.
  They will remember whichever loaded first. Featuring three to five standouts, at least one at
  production scale, each with problem → ownership → architecture → stack → **metric** → evolution,
  and demoting the rest to a chapter index, does the synthesis for them and demonstrates editorial
  judgement, which is the thing being graded. Completeness is not the rubric; selection is.

- **Which design-doc section most separates staff from senior, and what GPU-specific content
  belongs in it?** *Answer:* Alternatives considered, closely followed by Non-goals. A senior doc
  often omits Alternatives entirely or lists options with no verdict, which reads as a literature
  review. The GPU-specific content is the partitioning table — passthrough / MIG / time-slicing /
  MPS across isolation, attribution cleanliness, utilisation and stranding, with real figures such
  as 7 × `1g.10gb` summing to 0.858 of an A100 on a memory basis — closed with a stated choice tied
  to a numbered Goal, plus the condition under which you would reverse it. The reversal condition
  is what proves the verdict was reasoned rather than defaulted.

- **State the time-slicing attribution hole, its mechanism, and how you quantify it.** *Answer:*
  under time-slicing, several pods share one physical device and the profiling counter is scoped to
  the *device*, not to the CUDA context resident when it was sampled — so all N sharers read the
  same aggregate value and no code recovers N per-tenant numbers from one device number; the
  information was never produced. Quantify it with the fair-share estimator: for a device at
  utilisation `u` shared by `N` pods with true shares `sᵢ`, the error is `errᵢ = u × (1/N − sᵢ)`.
  Σ errᵢ = 0, so the error is purely redistributive — the device total is exactly right and only
  the split is wrong — and `|errᵢ| ≤ u`. Aggregate to an exposure fraction:
  `(G_s × ū_s) / (G × ū_all)`, which on a 64-GPU fleet with 24 time-sliced devices at 0.91
  utilisation against 40 exact devices at 0.62 is 21.84/46.64 = 46.8%, or about $63.8k/month at
  $4.00/GPU-hour. It belongs in Risks, first, with three mitigations: label every series with its
  attribution rung, bill from the allocated ledger which has no exposure, and pursue per-PID
  attribution to move those dollars from estimate to measurement.

- **What exactly are you entitled to claim when you say "FOCUS-aligned," and what must you add?**
  *Answer:* you may claim conformance to a named version — FOCUS 1.4 is the ratified release as of
  June 2026 — by populating the mandatory spine (`ChargeCategory`, `ChargePeriodStart/End`,
  `ResourceId`, `ServiceName`), the four cost columns with `EffectiveCost` as your allocation
  basis, `ConsumedQuantity` in GPU-Hours, and the split-cost-allocation columns from 1.3
  (`AllocatedResourceId`, `AllocatedMethodId`, `AllocatedMethodDetails`, whose per-entry
  `AllocatedRatio` values must sum to 1 across all allocated charges from one origin charge). You
  must add two `x_` extensions and say why: the standard has no utilisation ledger at all — every
  cost and quantity column including the new `Allocated*` ones is allocated or billed, and
  `ConsumedQuantity` is provider-metered rather than a measurement of device work — and it carries
  no confidence band on a declared split, which is fine for MIG (`exact`) and not fine for
  time-slicing.

- **Why is a specific error bound with no derivation worse than no number at all?** *Answer:*
  because credibility is not additive, it is multiplicative. A reviewer who cannot check a number
  has to decide whether to trust the author, and a precise-looking figure with no visible method is
  the signature of an invented one — so the doubt does not stay local, it propagates to every other
  figure in the document. Showing three lines of algebra and the aggregation converts the same
  number from a claim into an analysis the reader can disagree with on method, which is exactly the
  disagreement you want.

- **Name three rendering constraints that decide whether your published portfolio actually works.**
  *Answer:* (i) Liquid is disabled site-wide, so `{%` tags must not appear anywhere, including
  inside code fences. (ii) The stock Pages theme does not render mermaid, so a mermaid diagram
  appears as raw source on the published site while looking correct in the GitHub file view — ASCII
  in fenced blocks is the only form that renders identically in both. (iii) Relative links to a
  directory need the explicit `README.md` suffix or they 404 on the built site while working on
  github.com. All three are silent failures, which is why the only reliable test is to build the
  site and read it on a phone.

- **What is the brag-doc for, and why is it the one document you do not publish?** *Answer:* it is
  a running one-page log of outcomes in the shape *shipped X → moved metric Y from A to B →
  unblocked Z*, with a hard rule that a line without a metric does not go in. It is the raw
  material you mine for lesson 07's narration and lesson 08's STAR bank, and the fastest way to
  reload your own numbers before a loop. You do not publish it because its bar is completeness
  rather than polish — it is a working log, not a narrative — and because publishing it removes the
  incentive to keep it honest. What you publish is its distillate: the three or four strongest
  numbers, in the README's numbers table.

## Connections & what's next

This lesson is the output stage of lesson 02's synthesis: the four-verb thesis becomes the README's
subtitle and value-chain table, the featured chapters become the top of the index, and the
rejected-alternatives bank becomes the design doc's Alternatives section with its verdicts. It
reaches forward too — the Alternatives and Risks sections are exactly what gets drilled out loud in
lesson 05's design prompts, and the README's chapter narration is what lesson 07 practises
delivering conversationally. The published repo and design doc are also the substrate lesson 04
builds on: the flagship post is drawn from the same Context and Risks material, aimed at a public,
reach-oriented audience instead of a review board.

Next: **lesson 04** takes this written portfolio and adds the public contrarian proof — the
flagship post and the narrated demo — that leads every application from here forward.

## References & further reading

**Primary sources**

- FinOps Foundation — FOCUS specification — https://focus.finops.org/ · https://focus.finops.org/focus-specification/ — read for: the column definitions your alignment claim points at, the `x_` custom-column convention, and the split-cost-allocation columns introduced in 1.3 with the `AllocatedRatio`-sums-to-1 constraint. FOCUS 1.4 is the ratified release as of June 2026.
- NVIDIA `dcgm-exporter`, `etc/default-counters.csv` — https://github.com/NVIDIA/dcgm-exporter/blob/main/etc/default-counters.csv — read for: the mechanism behind design-doc risk R2 — the honest metric ships commented out.
- NVIDIA `dcgm-exporter` CLI and Helm chart — https://github.com/NVIDIA/dcgm-exporter — read for: `--kubernetes` (default `false`), `--collect-interval` (default 30000 ms, which is your integration constant), the pod-resources socket path, and the `SYS_ADMIN` capability the profiling fields require.
- Prometheus — `scrape_config` and `honor_labels` — https://prometheus.io/docs/prometheus/latest/configuration/configuration/ — read for: the `exported_` renaming that produces design-doc risk R3, and why it is a scrape-config property rather than an exporter property.
- NVIDIA — Multi-Instance GPU user guide — https://docs.nvidia.com/datacenter/tesla/mig-user-guide/ — read for: the profile geometry behind the Alternatives table, including why 7 × `1g.10gb` does not sum to a whole card on a memory basis.
- Meta — *Revisiting Reliability in Large-Scale Machine Learning Research Clusters* (arXiv:2410.21680) — https://arxiv.org/abs/2410.21680 — read for: the measured 7.9-hour MTTF at 1,024 GPUs that justifies the survivability layer in the platform diagram; note the larger-scale figures are the paper's projections.

**Course-internal sources — where the worked doc's numbers come from**

- `platform-eng/modules/05-gpu-observability/lessons/01-lie-and-truth.md` — field 203's exact semantics and the two collection paths behind the Context section.
- `platform-eng/modules/05-gpu-observability/lessons/08-capstone-allocated-vs-utilised.md` — the 576/231.4 GPU-hour worked fleet, the 57.3% idle share, the `sum_over_time` correction, and the five validation checks reproduced in Rollout.
- `platform-eng/modules/04-gpu-on-kubernetes/lessons/10-capstone-per-pod-attribution.md` — the two-loop architecture, the share model and its bases, identities A and B, and the exposure-fraction derivation used in R1.
- `platform-eng/modules/11-gpu-cost-economics/lessons/10-focus-spec.md` — the FOCUS column list, the 1.3/1.4/1.5 version history, and the two gaps the `x_` extensions close.

**Real-world engineering writing**

- Julia Evans — "Get your work recognized: write a brag document" — https://jvns.ca/blog/brag-documents/ — read for: the working-log format behind §7, and specifically why the document exists for *you* rather than for a reader.
- "Design Docs at Google" — https://www.industrialempathy.com/posts/design-docs-at-google/ — read for: why Non-goals and Alternatives exist as mandatory sections in a large engineering organisation, and what reviewers actually do with them.

**Not relied upon**

- Company blog posts describing individual organisations' RFC processes and public design-doc
  corpora were consulted for the structural habits noted in §8 and Real-world use cases. The
  specific practices are described as general patterns rather than as verified claims about any
  named company's current process, since those pages were not fetched during this revision.
