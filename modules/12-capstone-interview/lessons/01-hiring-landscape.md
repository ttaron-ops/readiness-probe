---
lesson: 01
title: "The GPU-infra hiring landscape (2025-2026)"
module: 12
concept: "decode real interview loops"
status: not-started
est_time: "3 hrs"
artifacts: ["per-company loop map + artifact-to-signal matrix + personal sequencing plan"]
---

# The GPU-infra hiring landscape (2025-2026)

## Why this matters
You are not interviewing for "a platform job." You are interviewing for a narrow, hot, and structurally different market: GPU-fleet operators who are hiring against 250k-GPU realities, not CRUD backends. The loops at CoreWeave, Lambda, Together, Modal, and Nebius look nothing like the FAANG loop you already know how to beat — most of them have deliberately stripped leetcode and replaced it with "prove you have actually operated GPUs at scale." If you prep the wrong loop shape, you either over-index on DSA the neoclouds don't ask, or you walk into a live Linux round at Lambda expecting a whiteboard and get handed an SSH prompt.

The second-order stake is sequencing. You have a 12-15 month runway and a portfolio of 11 artifacts that map almost 1:1 onto neocloud job descriptions. That is a genuine, unusual edge — your gpu-cost-operator and your "dashboard is lying" writeup are literally the syllabus these companies interview against. But that edge is worth the most against the no-DSA tier and the least against NVIDIA's structured, still-DSA-flavored onsite. Knowing which door your artifacts open, and in what order to knock, is the difference between a scattered 15-loop grind and a targeted campaign where your first three loops are the ones you're most likely to convert.

This lesson decodes the real 2025-2026 loops company by company so the rest of the module preps the right things.

## What's new here
- **Skip** (you already own this): generic behavioral/STAR mechanics, how to negotiate, resume formatting, the FAANG system-design rubric, DSA prep strategy. You are sharp on all of it.
- **Skip**: "what is a neocloud" — you built against these stacks for 11 modules.
- **New**: the *specific* loop shape per company (screen types, round counts, take-home presence), which are genuinely no-DSA, and the exact JD language that tells you what proof they want.
- **New**: the 2025-2026 shifts at AI labs (in-person onsites rising, AI-tools-permitted coding rounds at OpenAI, take-home revival, reference weighting).
- **New**: an artifact-to-signal matrix — which of your 11 artifacts is the *killer proof* for each employer — and a personal sequencing plan that leads with the loops your portfolio was accidentally built for.

## Core notes

### CoreWeave — public since Mar 2025, 250k+ GPUs
Roles: Sr GPU Infrastructure SWE, Staff SWE (Compute Architecture), Sr SWE (Kubernetes / Network / Observability), SRE. The JD language is the tell: you "own an area" and are accountable for **"measurable improvements to latency/throughput/reliability across services."** They want engineers who build gRPC/REST services in **Go or Python** that drive Kubernetes, who write **custom controllers and operators**, who can stand up a GPU performance-testing/validation platform, and who live in Grafana/Prometheus. The SRE track adds zero-touch ops, formal RCA, and on-call.
- **Loop shape**: recruiter → technical screen (live coding, optimization-flavored, not pure leetcode) → K8s/GPU systems design → domain deep-dive → behavioral/staff-signal.
- **Your killer proof**: gpu-cost-operator (02) is a custom controller against a real fleet — this *is* the "custom controllers and operators" bullet — backed by gpu-cost-exporter (01) as the Prometheus half.

### Lambda — the cleanest no-DSA loop in the market
Team you want: **"Fleet Foundations — GPU host lifecycle & machine management at scale."** Loop: recruiter → HM technical (resume deep-dive + domain) → **live Linux round: you SSH into a real instance and answer live** → domain synthesis. Explicitly **no leetcode**. The live Linux round rewards someone who has actually debugged a sick GPU host — `nvidia-smi`, dmesg, Xid errors, a wedged driver, a NUMA-misplaced process — not someone who memorized graph algorithms.
- **Your killer proof**: survive-a-failure training lab (08) and the per-pod attribution + failure-mode log (04). The failure-mode log is exactly the mental model the live Linux round probes.

### Together AI — the deepest technical bar
Expect low-level CUDA questions, distributed-inference design, and real-time low-latency concerns. Canonical prompt: **"design an inference system serving 100+ models with efficient GPU sharing."** The classic failure mode is hand-waving KV cache and batching — if you can't reason about continuous batching, KV-cache memory pressure, and the sharing tradeoff (MIG vs time-slice vs multi-model-on-one-GPU), you stall here.
- **Your killer proof**: cost-per-million-tokens (07) is the direct antidote — it forces you to have already reasoned about batching, KV cache, and per-token economics.

### Modal — serverless GPU, Python-first
Custom scheduler, scale-to-zero. Prompts skew toward **cold-start latency, scale-to-zero economics, and scheduler internals** (how do you place a GPU workload in seconds, how do you avoid paying for idle accelerators). Python-first shop.
- **Your killer proof**: capex-vs-cloud (10) and Kueue showback (06) show you think about idle cost and scheduling; be ready to reason about cold-start on top.

### NVIDIA — most structured, still DSA-flavored
Loop: recruiter → peer screen (resume deep-dive + **live coding with optimization follow-ups**) → onsite of **3-5 rounds** (coding, system design, domain deep-dive, behavioral STAR). This is the one tier where classic DSA prep still pays — treat it as the "keep leetcode warm" employer.

### Nebius — Go/K8s/GPU-fleet, same stack
Roles: Sr SRE, Staff SWE, and **"Forward Deployed Engineer — Physical AI Cloud Platform."** Same Go/Kubernetes/GPU-fleet toolchain as CoreWeave and Lambda; your artifacts transfer directly.

### AI labs (OpenAI / Anthropic / Mistral) — 2025-2026 shifts
- **In-person onsites are rising** (~24% → ~38% of loops).
- **OpenAI permits AI tools in coding rounds** — the evaluated signal is *narrate your reasoning; judgment beats raw output*. Talking through why you accept/reject the model's suggestion is the round.
- **Take-homes are back**: 4-8h on realistic problems, not toy puzzles.
- **References matter**: line up 2-3 recent managers.

### Consolidated modal loop (neoclouds)
recruiter → technical/coding screen (live, optimization-flavored, **no pure leetcode at neoclouds**) → GPU/infra system-design → domain deep-dive / debug-the-incident → behavioral/staff-signal → *sometimes* a 4-8h take-home.

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
Lead with the **neoclouds** (CoreWeave, Lambda, Nebius, Modal, Together). They are no-DSA-or-optimization-flavored, and your 11 artifacts are effectively their interview syllabus — you convert highest here and the prep doubles as portfolio polish. Run 2-3 of these first, both to bank offers and to rehearse the domain narrative live. Treat **NVIDIA and big-tech ML-infra** as the DSA-flavored tier: worth it, but gate them behind a few weeks of kept-warm leetcode, and schedule them *after* you've done a couple neocloud loops so your domain story is already sharp. Keep AI-lab loops flexible for the take-home and reference logistics.

## Worked example
Rehearsal — Lambda live Linux round. The interviewer SSHes you into a box: "This node's training job is running at half the throughput it did yesterday. Go." You don't reach for an algorithm; you reach for your failure-mode log (04). Out loud: "First `nvidia-smi` — check utilization, ECC error counts, and whether a GPU fell off the bus (Xid). Then `dmesg -T | grep -i xid` for hardware faults. Then confirm the process is on the right NUMA node with `nvidia-smi topo -m` and `numactl` — a common half-throughput cause is a data-loader pinned across the wrong socket. Then check thermals and power capping via `nvidia-smi -q -d POWER,TEMPERATURE`; a throttled clock halves throughput silently. Then PCIe/NVLink link width in case a link renegotiated down." That narration — hypothesis, command, expected reading, next branch — is the entire signal. It maps directly onto the survive-a-failure lab (08) you already built, which is why you lead your Lambda prep with it.

## Practice
Feed these into the capstone deliverable at [GPU platform capstone](../practice/gpu-platform-capstone/README.md):
1. Build your own version of the company map table for the 5-7 employers you'll actually apply to, with a live-JD link for each and the exact JD phrase that reveals what proof they want.
2. For each artifact (01-11), write one sentence naming which employer/round it is the killer proof for. This is your artifact-to-signal matrix — commit it to the capstone README.
3. Write your sequencing plan: name your first 3 target loops (neoclouds), the DSA-tier loops and when you'll gate them, and the reference list (2-3 recent managers) for the AI-lab tier.
4. Draft your one-paragraph answer to Together's "design an inference system serving 100+ models with efficient GPU sharing," explicitly naming KV cache, continuous batching, and the MIG-vs-time-slice sharing tradeoff.

## Self-check
- Which employer has the cleanest explicitly-no-leetcode loop, and what is its signature round? **Answer:** Lambda — a live Linux round where you SSH into a real instance and debug/answer live; it rewards hands-on GPU-host operational instinct, mapped by your failure-mode log (04) and survive-a-failure lab (08).
- Why should you sequence neocloud loops before NVIDIA/big-tech? **Answer:** Neoclouds are no-DSA / optimization-flavored and your 11 artifacts are effectively their interview syllabus, so you convert highest there; NVIDIA is the DSA-flavored tier best gated behind kept-warm leetcode and scheduled after your domain narrative is rehearsed live — and banking early neocloud offers gives leverage into the harder-prep tier.
- What is the classic failure mode in a Together AI inference-design round, and which artifact inoculates you? **Answer:** Hand-waving KV cache and batching; cost-per-million-tokens (07) forces you to have already reasoned about continuous batching, KV-cache memory pressure, and the GPU-sharing tradeoff.

## Resources
- CoreWeave Sr GPU Infrastructure SWE JD: https://www.welcometothejungle.com/en/companies/coreweave/jobs
- CoreWeave Kueue engineering blog: https://www.coreweave.com/blog/kueue-a-kubernetes-native-system-for-ai-training-workloads
- Lambda (Fleet Foundations / about): https://lambda.ai/about
- NVIDIA interview process: https://www.finalroundai.com/blog/nvidia-interview-process and https://www.tryexponent.com/blog/nvidia-interview-process
- AI-engineering interview trends field guide: https://github.com/alexeygrigorev/ai-engineering-field-guide

[🎓 12 — Capstone & interview preparation](../README.md)
