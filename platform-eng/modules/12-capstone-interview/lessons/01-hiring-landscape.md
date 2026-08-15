---
lesson: 01
title: "The GPU-infra hiring landscape (2025-2026)"
module: 12
concept: "decode real interview loops"
status: not-started
est_time: "4 hrs"
prev: null
next: "02-capstone-synthesis.md"
artifacts: ["per-company loop map + artifact-to-signal matrix + personal sequencing plan"]
sources: 10
---

# The GPU-infra hiring landscape (2025-2026)

## Where this fits
This lesson **opens** module 12 — Track C, the continuous capstone-assembly and interview-ramp
track that has been running since roughly month 4 and now takes over as the primary focus for
months 9-15. Everything before this point in the course was building: eleven modules of Go,
Linux internals, Kubernetes controllers, host topology, GPU hardware, observability, scheduling,
inference serving, distributed training, networking, bare-metal lifecycle, and cost economics.
This lesson is the pivot from *building the artifacts* to *deploying them* — it is where you stop
asking "what should I build next" and start asking "which door does what I already built open
fastest." Everything downstream in this module — the synthesis in lesson 02, the portfolio build
in lesson 03, the drills in lessons 05-09 — depends on knowing the shape of the loops you're
prepping for, which is exactly what this lesson maps.

## Why this matters
You are not interviewing for "a platform job." You are interviewing for a narrow, hot, and
structurally different market: GPU-fleet operators who are hiring against 250k-GPU realities, not
CRUD backends. The loops at CoreWeave, Lambda, Together, Modal, and Nebius look nothing like the
FAANG loop you already know how to beat — most of them have deliberately stripped leetcode and
replaced it with "prove you have actually operated GPUs at scale." If you prep the wrong loop
shape, you either over-index on DSA the neoclouds don't ask, or you walk into a live Linux round
at Lambda expecting a whiteboard and get handed an SSH prompt.

The second-order stake is sequencing. You have a 12-15 month runway and a portfolio of 11
artifacts that map almost 1:1 onto neocloud job descriptions. That is a genuine, unusual edge —
your gpu-cost-operator and your "dashboard is lying" writeup are literally the syllabus these
companies interview against. But that edge is worth the most against the no-DSA tier and the
least against NVIDIA's structured, still-DSA-flavored onsite. Knowing which door your artifacts
open, and in what order to knock, is the difference between a scattered 15-loop grind and a
targeted campaign where your first three loops are the ones you're most likely to convert.

There is also a real economic stake behind the loop shape itself: GPU-fleet operators run
capital-intensive businesses on thin margins — every idle GPU-hour is a line item measured in
real dollars, not an abstraction. That is *why* the loops look the way they do. A company burning
cash on 250k GPUs cannot afford to hire someone who needs six months to become dangerous; it hires
against direct evidence that you already know how to cut idle cost, debug a wedged host, or reason
about KV-cache pressure on day one. Understanding that economic pressure is what makes the rest of
this lesson's loop-by-loop detail make sense instead of reading as arbitrary company trivia.

This lesson decodes the real 2025-2026 loops company by company so the rest of the module preps
the right things.

## What's new here (calibration)
- **Skip** (you already own this): generic behavioral/STAR mechanics, how to negotiate, resume
  formatting, the FAANG system-design rubric, DSA prep strategy. You are sharp on all of it.
- **Skip**: "what is a neocloud" — you built against these stacks for 11 modules.
- **New**: the *specific* loop shape per company (screen types, round counts, take-home
  presence), which are genuinely no-DSA, and the exact JD language that tells you what proof they
  want.
- **New**: the 2025-2026 shifts at AI labs (in-person onsites rising, AI-tools-permitted coding
  rounds at OpenAI, take-home revival, reference weighting).
- **New**: an emerging role shape — the **forward-deployed GPU-platform engineer** (Nebius'
  "Physical AI Cloud Platform" track) — that blends fleet-ops depth with direct customer-facing
  deployment work, a hybrid you should recognize if it shows up elsewhere.
- **New**: an artifact-to-signal matrix — which of your 11 artifacts is the *killer proof* for
  each employer — and a personal sequencing plan that leads with the loops your portfolio was
  accidentally built for.

## Core concepts

### CoreWeave — public since Mar 2025, 250k+ GPUs
Roles: Sr GPU Infrastructure SWE, Staff SWE (Compute Architecture), Sr SWE (Kubernetes / Network /
Observability), SRE. The JD language is the tell: you "own an area" and are accountable for
**"measurable improvements to latency/throughput/reliability across services."** They want
engineers who build gRPC/REST services in **Go or Python** that drive Kubernetes, who write
**custom controllers and operators**, who can stand up a GPU performance-testing/validation
platform, and who live in Grafana/Prometheus. The SRE track adds zero-touch ops, formal RCA, and
on-call. The live CoreWeave Sr GPU Infrastructure SWE posting confirms this exact language,
verbatim — it is not paraphrase or interview-prep-site color, it is the literal rubric.
- **Loop shape**: recruiter → technical screen (live coding, optimization-flavored, not pure
  leetcode) → K8s/GPU systems design → domain deep-dive → behavioral/staff-signal.
- **Your killer proof**: gpu-cost-operator (02) is a custom controller against a real fleet —
  this *is* the "custom controllers and operators" bullet — backed by gpu-cost-exporter (01) as
  the Prometheus half.

### Lambda — the cleanest no-DSA loop in the market
Team you want: **"Fleet Foundations — GPU host lifecycle & machine management at scale."** Loop:
recruiter → HM technical (resume deep-dive + domain) → **live Linux round: you SSH into a real
instance and answer live** → domain synthesis. Explicitly **no leetcode**. The live Linux round
rewards someone who has actually debugged a sick GPU host — `nvidia-smi`, dmesg, Xid errors, a
wedged driver, a NUMA-misplaced process — not someone who memorized graph algorithms.
- **Your killer proof**: survive-a-failure training lab (08) and the per-pod attribution +
  failure-mode log (04). The failure-mode log is exactly the mental model the live Linux round
  probes.

### Together AI — the deepest technical bar
Expect low-level CUDA questions, distributed-inference design, and real-time low-latency
concerns. Canonical prompt: **"design an inference system serving 100+ models with efficient GPU
sharing."** A field-guide account of Together's process corroborates this exact prompt and names
the specific depth it expects: reasoning about **PagedAttention**, **FlashAttention**, and
**quantization** as first-class tools, not footnotes. The classic failure mode is hand-waving KV
cache and batching — if you can't reason about continuous batching, KV-cache memory pressure, and
the sharing tradeoff (MIG vs time-slice vs multi-model-on-one-GPU), you stall here.
- **Your killer proof**: cost-per-million-tokens (07) is the direct antidote — it forces you to
  have already reasoned about batching, KV cache, and per-token economics.

### Modal — serverless GPU, Python-first
Custom scheduler, scale-to-zero. Prompts skew toward **cold-start latency, scale-to-zero
economics, and scheduler internals** (how do you place a GPU workload in seconds, how do you avoid
paying for idle accelerators). Python-first shop. Modal's own cold-start documentation is worth
knowing cold (pun intended) before this round: GPU memory snapshotting is a documented,
production technique that has cut cold-start latency by roughly an order of magnitude in
published cases — mid-sized LLM cold starts falling from on the order of a minute-plus down to
single-digit seconds by snapshotting warmed-up GPU memory state instead of re-initializing from
scratch. You don't need Modal's exact internals memorized, but you should be able to explain *why*
that technique works (skip re-loading weights and re-warming CUDA context by resuming from a
memory image) — that's the shape of the answer this round wants.
- **Your killer proof**: capex-vs-cloud (10) and Kueue showback (06) show you think about idle
  cost and scheduling; be ready to reason about cold-start on top.

### NVIDIA — most structured, still DSA-flavored
Loop: recruiter → peer screen (resume deep-dive + **live coding with optimization follow-ups**) →
onsite of **3-5 rounds** (coding, system design, domain deep-dive, behavioral STAR). This is the
one tier where classic DSA prep still pays — treat it as the "keep leetcode warm" employer. A
first-person SDE-2 account (anecdotal, one data point, but a real candidate's real loop) describes
exactly this shape: a live-coding screen followed by a multi-round onsite mixing coding, design,
and behavioral — useful as a gut-check against the aggregate process descriptions, not as a
guarantee your loop will match it exactly.

### Nebius — Go/K8s/GPU-fleet, same stack, plus a new hybrid role
Roles: Sr SRE, Staff SWE, and **"Forward Deployed Engineer — Physical AI Cloud Platform."** The
core Sr SRE / Staff SWE tracks run the same Go/Kubernetes/GPU-fleet toolchain as CoreWeave and
Lambda; your artifacts transfer directly. The Forward Deployed Engineer track is worth flagging on
its own: it's a live posting for a role that blends deep fleet/platform knowledge with
direct, on-site deployment work for physical-AI customers — a hybrid of platform engineering and
forward-deployed/solutions engineering. If you see this shape elsewhere in your search (it is
becoming a recognizable pattern at infra companies serving robotics/physical-AI customers), read
it as "platform depth, but you also have to stand in front of a customer and make it work in their
environment" — a different interview emphasis (more scenario/communication-heavy) than a pure
platform-SWE loop.

### AI labs (OpenAI / Anthropic / Mistral) — 2025-2026 shifts
- **In-person onsites are rising** (~24% → ~38% of loops).
- **OpenAI permits AI tools in coding rounds** — the evaluated signal is *narrate your reasoning;
  judgment beats raw output*. Talking through why you accept/reject the model's suggestion is the
  round.
- **Take-homes are back**: 4-8h on realistic problems, not toy puzzles.
- **References matter**: line up 2-3 recent managers.

### Consolidated modal loop (neoclouds)
recruiter → technical/coding screen (live, optimization-flavored, **no pure leetcode at
neoclouds**) → GPU/infra system-design → domain deep-dive / debug-the-incident →
behavioral/staff-signal → *sometimes* a 4-8h take-home.

### The map

| Company | Role titles | Loop shape | No-DSA? | Killer artifact |
|---|---|---|---|---|
| CoreWeave | Sr GPU Infra SWE, Staff SWE Compute Arch, Sr K8s/Net/Obs, SRE | screen → K8s/GPU design → domain → behavioral | Mostly (optimization, not leetcode) | gpu-cost-operator (02) + exporter (01) |
| Lambda | Fleet Foundations (host lifecycle) | HM tech → **live Linux SSH** → domain synthesis | **Yes, explicit** | survive-a-failure (08) + failure-mode log (04) |
| Together AI | Inference / systems eng | deep CUDA + distributed-inference design | Mostly (systems-heavy) | cost-per-million-tokens (07) |
| Modal | Platform / scheduler eng | cold-start / scale-to-zero / scheduler internals | Mostly | capex-vs-cloud (10) + Kueue showback (06) |
| NVIDIA | SWE (multiple) | peer screen (live coding) → 3-5× onsite | **No — DSA-flavored** | topology teardown (02b) + network read (09) |
| Nebius | Sr SRE, Staff SWE, Forward Deployed Eng | Go/K8s/GPU-fleet screen → design → domain | Mostly | gpu-cost-operator (02) + FOCUS synthesis (11) |
| AI labs | ML-infra / platform | in-person onsite ↑, AI-tools-permitted coding, 4-8h take-home, references | Partly | "dashboard is lying" (05) + survive-a-failure (08) |

### Your personal sequencing strategy
Lead with the **neoclouds** (CoreWeave, Lambda, Nebius, Modal, Together). They are
no-DSA-or-optimization-flavored, and your 11 artifacts are effectively their interview syllabus —
you convert highest here and the prep doubles as portfolio polish. Run 2-3 of these first, both to
bank offers and to rehearse the domain narrative live. Treat **NVIDIA and big-tech ML-infra** as
the DSA-flavored tier: worth it, but gate them behind a few weeks of kept-warm leetcode, and
schedule them *after* you've done a couple neocloud loops so your domain story is already sharp.
Keep AI-lab loops flexible for the take-home and reference logistics.

## Perspectives

**Candidate's view — which loop converts your artifacts fastest.** From where you sit, the
question isn't "which company is most prestigious," it's "which loop turns my existing 11
artifacts into a highest-probability offer with the least additional prep." That reframes the
whole search: Lambda's live Linux round is closer to a rehearsal of your failure-mode log than a
test, and CoreWeave's system-design round is closer to a defense of the operator you already
shipped than an open-ended whiteboard. Ranking companies by prestige and ranking them by
artifact-conversion-rate produce different orderings — sequencing on the second one is the whole
point of this lesson.

**Hiring-manager's view — what JD language literally signals, and why neoclouds stripped DSA.**
A hiring manager writing "measurable improvements to latency/throughput/reliability across
services" is not being aspirational — that's a rubric line an interviewer will score you against
directly, because it's what the role does all day. Neoclouds strip DSA for the same reason
capital-intensive fleets pressure-test differently than product companies: a company burning
cash on GPU-hours can't afford six months of ramp-up. They need direct evidence you've operated
infrastructure like theirs, which a graph-traversal exercise doesn't provide and a live SSH
session does.

**Market-economics view — thin margins demand engineers who cut idle cost day one.** GPU-fleet
operators sit on enormous fixed capital (a single H100 node can run into six figures) recovered
through utilization over years, on margins that are thin relative to the capital at risk. Every
idle GPU-hour, every mis-scheduled job, every slow incident response is a direct, quantifiable
cost. That is the economic engine behind every loop in this lesson's map: they interview for
people who already think in GPU-hours and dollars, which is precisely what modules 06 and 11 of
this course trained you to do.

**Company-lifecycle view — loop shape tracks company maturity.** CoreWeave, public since March
2025 with formalized leveling and process, runs the most structured neocloud loop. Lambda and
Together, still in growth-stage financing, run leaner, faster, more improvisational loops (a
single live-Linux round, a systems-design conversation) because they can't afford — and don't yet
need — heavyweight process. NVIDIA, a decades-old incumbent, runs the most standardized,
most-rounds loop of the set. Reading a company's loop shape against its lifecycle stage helps you
predict what an unfamiliar employer's process will look like even before you find their exact
loop map.

## Real-world use cases
- **CoreWeave Senior GPU Infrastructure SWE job posting** —
  https://www.welcometothejungle.com/en/companies/coreweave/jobs/gpu-infrastructure-software-engineer_sunnyvale_y27bo3df
  — what it shows: the literal, live JD language ("custom controllers and operators," gRPC/REST in
  Go/Python, hardware validation platform) that this lesson's Core concepts section quotes almost
  verbatim, confirming the JD phrasing is a real current rubric, not paraphrase.
- **Nebius "Forward Deployed Engineer — Physical AI Cloud Platform" job posting** —
  https://himalayas.app/companies/nebius/jobs/forward-deployed-engineer-physical-ai-cloud-platform
  — what it shows: a real, currently-live posting for the hybrid platform/forward-deployed role
  described above — evidence this role shape exists in the market right now, not a hypothetical.
- **Together AI interview-process field guide** — https://www.techinterview.org/companies/together-ai/
  — what it shows: independent corroboration of the "design an inference system for 100+ models
  with GPU sharing" prompt, and that PagedAttention, FlashAttention, and quantization are named
  as expected depth for that round.
- **Modal cold-start documentation** — https://modal.com/docs/guide/cold-start — what it shows:
  the primary-source technical detail (GPU memory snapshotting) behind Modal's cold-start
  interview prompts — the actual engineering technique you should be able to reason about, not
  just the fact that "cold start matters."

## Worked example
Rehearsal — Lambda live Linux round. The interviewer SSHes you into a box: "This node's training
job is running at half the throughput it did yesterday. Go." You don't reach for an algorithm; you
reach for your failure-mode log (04). Out loud: "First `nvidia-smi` — check utilization, ECC error
counts, and whether a GPU fell off the bus (Xid). Then `dmesg -T | grep -i xid` for hardware
faults. Then confirm the process is on the right NUMA node with `nvidia-smi topo -m` and
`numactl` — a common half-throughput cause is a data-loader pinned across the wrong socket. Then
check thermals and power capping via `nvidia-smi -q -d POWER,TEMPERATURE`; a throttled clock halves
throughput silently. Then PCIe/NVLink link width in case a link renegotiated down." That narration
— hypothesis, command, expected reading, next branch — is the entire signal. It maps directly onto
the survive-a-failure lab (08) you already built, which is why you lead your Lambda prep with it.

A second rehearsal, this time for Together AI's canonical prompt: "design an inference system
serving 100+ models with efficient GPU sharing." A strong answer names the tradeoff explicitly:
"For 100+ models on shared GPUs I'd default to time-slicing or dynamic multi-model packing over
MIG, because MIG's static partitions don't rebalance as model traffic shifts, and 100+ models
means constant rebalancing. I'd batch requests continuously (not statically) to keep GPUs busy
between variable-length generations, and size KV-cache budget per model against expected
concurrency — that's usually the actual bottleneck before compute is, which is exactly what my
cost-per-million-tokens work (07) forced me to quantify." Naming PagedAttention/FlashAttention by
name, and being ready to say why continuous batching beats static batching for a long-tail
100-model traffic mix, is the difference between passing and stalling here.

## Practice
Feed these into the capstone deliverable at
[GPU platform capstone](../practice/gpu-platform-capstone/README.md):
1. Build your own version of the company map table for the 5-7 employers you'll actually apply
   to, with a live-JD link for each and the exact JD phrase that reveals what proof they want.
2. For each artifact (01-11), write one sentence naming which employer/round it is the killer
   proof for. This is your artifact-to-signal matrix — commit it to the capstone README.
3. Write your sequencing plan: name your first 3 target loops (neoclouds), the DSA-tier loops and
   when you'll gate them, and the reference list (2-3 recent managers) for the AI-lab tier.
4. Draft your one-paragraph answer to Together's "design an inference system serving 100+ models
   with efficient GPU sharing," explicitly naming KV cache, continuous batching, PagedAttention /
   FlashAttention, and the MIG-vs-time-slice sharing tradeoff.
5. For Modal-style employers, write two sentences explaining *why* GPU memory snapshotting cuts
   cold-start latency — not just that it does — so you can defend the mechanism live, not just
   cite the number.

## Common pitfalls
1. **Treating "no LeetCode" as "no technical bar."** Lambda's live Linux round is, if anything,
   *harder* to fake than a DSA round — you can't memorize your way through a live SSH session on a
   real box with a real fault. "No-DSA" means a different technical bar, not a lower one; prep
   accordingly.
2. **Applying to NVIDIA/big-tech first because it feels "safe."** This burns your best-prepped
   shot on the exact tier where your 11 artifacts help the least (NVIDIA's loop is DSA-flavored,
   not domain-narrative-flavored). Lead with the neoclouds where your portfolio is the syllabus,
   and treat NVIDIA as a later, separately-prepped tier.
3. **Assuming JD buzzwords are filler.** "Custom controllers and operators," "measurable
   improvements to latency/throughput/reliability" — these are not resume-bait, they are the
   literal rubric an interviewer scores against. Read every target JD as a checklist, not
   marketing copy.
4. **Not sequencing — treating 5-7 target companies as parallel and interchangeable.** A
   scattergun of simultaneous applications wastes your best-prepared narrative on whichever
   recruiter calls back first, rather than on the company where it converts highest. Build a
   deliberate campaign order (this lesson's sequencing strategy) instead.
5. **Underestimating the emerging hybrid roles.** A posting like Nebius' Forward Deployed Engineer
   looks like a normal platform role at a glance but actually interviews for a different skill mix
   (customer-facing scenario handling alongside fleet depth). Read the full JD, not just the
   title, before assuming a role's loop matches the standard platform-SWE shape.

## Self-check
- Which employer has the cleanest explicitly-no-leetcode loop, and what is its signature round?
  **Answer:** Lambda — a live Linux round where you SSH into a real instance and debug/answer
  live; it rewards hands-on GPU-host operational instinct, mapped by your failure-mode log (04)
  and survive-a-failure lab (08).
- Why should you sequence neocloud loops before NVIDIA/big-tech?
  **Answer:** Neoclouds are no-DSA / optimization-flavored and your 11 artifacts are effectively
  their interview syllabus, so you convert highest there; NVIDIA is the DSA-flavored tier best
  gated behind kept-warm leetcode and scheduled after your domain narrative is rehearsed live —
  and banking early neocloud offers gives leverage into the harder-prep tier.
- What is the classic failure mode in a Together AI inference-design round, and which artifact
  inoculates you?
  **Answer:** Hand-waving KV cache and batching; cost-per-million-tokens (07) forces you to have
  already reasoned about continuous batching, KV-cache memory pressure, and the GPU-sharing
  tradeoff — and Together's own interview field guide confirms PagedAttention, FlashAttention, and
  quantization are the expected depth.
- Why did neoclouds strip DSA from their loops instead of just shortening the loop?
  **Answer:** Because DSA doesn't measure the thing that actually matters to a capital-intensive,
  thin-margin GPU fleet: whether you can operate infrastructure like theirs on day one. They
  replaced it with direct evidence — live debugging, domain system design — because that's what
  the JD language ("measurable improvements to latency/throughput/reliability") is actually
  scoring.
- What technique should you be able to explain for a Modal-style cold-start round, and why does it
  work?
  **Answer:** GPU memory snapshotting — resuming a warmed-up container/GPU memory image instead of
  re-initializing weights and re-warming CUDA context from scratch, which is what cuts cold starts
  from roughly a minute-plus down to single-digit seconds in documented cases.

## Connections & what's next
This lesson produces the raw material — the loop map, the artifact-to-signal matrix, and the
sequencing plan — that every later lesson in this module consumes. Lesson 02 takes the same 11
artifacts this lesson maps to employers and threads them into a single narrative you'll actually
tell in the room. Lesson 03 turns that narrative into a written portfolio. Lessons 05-06 use the
company map to prioritize which system-design prompts (P1-P6) and debugging drills matter most for
your first few loops. Lesson 09's mock loop is scored against the exact loop shapes mapped here.
Next: **lesson 02** takes the artifact-to-signal matrix this lesson built and asks a different
question — not "which artifact fits which employer" but "what single thesis makes all 11 read as
one deliberate body of work."

## References & further reading

**Primary sources**
- CoreWeave Senior GPU Infrastructure SWE job posting: https://www.welcometothejungle.com/en/companies/coreweave/jobs/gpu-infrastructure-software-engineer_sunnyvale_y27bo3df
- Nebius Forward Deployed Engineer — Physical AI Cloud Platform job posting: https://himalayas.app/companies/nebius/jobs/forward-deployed-engineer-physical-ai-cloud-platform
- Modal cold-start performance documentation: https://modal.com/docs/guide/cold-start

**Real-world engineering blogs**
- CoreWeave Kueue engineering blog: https://www.coreweave.com/blog/kueue-a-kubernetes-native-system-for-ai-training-workloads
- NVIDIA SDE-2 interview experience (first-person, anecdotal): https://medium.com/@dadhichhardik26/nvidia-sde-2-interview-experience-offer-41d79116d478

**Deeper dives**
- Together AI interview-process field guide: https://www.techinterview.org/companies/together-ai/
- NVIDIA interview process overviews: https://www.finalroundai.com/blog/nvidia-interview-process and https://www.tryexponent.com/blog/nvidia-interview-process
- AI-engineering interview trends field guide: https://github.com/alexeygrigorev/ai-engineering-field-guide
- Lambda (Fleet Foundations / company background): https://lambda.ai/about

[🎓 12 — Capstone & interview preparation](../README.md)
