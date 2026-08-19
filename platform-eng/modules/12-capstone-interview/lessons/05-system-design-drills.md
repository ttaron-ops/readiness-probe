---
lesson: 05
title: "GPU system-design drills (P1–P6)"
module: 12
concept: "GPU system-design reps"
status: not-started
est_time: "6 hrs"
prev: "04-flagship-blog-demo.md"
next: "06-debugging-drills.md"
artifacts: ["gpu-system-design-drill-log", "p2-answer-skeleton"]
sources: 12
---

# GPU system-design drills (P1–P6)

## Where this fits

Lesson 04 got your flagship public and demo-ready — the artifact that proves you *built* something real, with an argument that survives a hostile reader. That proof answers "have you done this?" but not "can you design this live, cold, under a clock, for a prompt you have never seen?" That is a different muscle: synthesising everything from modules 01–11 into a fluent, unprompted, twenty-five-minute spoken design under interview pressure. This lesson drills that muscle directly, against the six canonical GPU-infra design prompts, as full run-throughs rather than checklists.

## Why this matters

You already design distributed systems in your sleep. What a GPU-fleet operator's design round actually tests is whether you carry the **hardware overlay** into the room: that a GPU is a shared, partitionable, failure-prone, absurdly expensive accelerator whose headline utilisation number lies, and whose interconnect is a first-class scheduling constraint. Generic answers — load balance, shard, cache, queue — score you as a strong web engineer, which is exactly the classification you are trying to escape.

The differentiator at this level is a *behaviour*, not a body of knowledge: you **volunteer scale, cost, failure modes and SLOs before the interviewer asks.** Weak candidates wait to be prompted ("have you thought about failure?"). Strong candidates open with "at roughly a thousand GPUs, at a couple of dollars per GPU-hour, here is the SLO I am protecting and the failure modes that dominate." The reason this matters mechanically is that the interviewer is filling in a scoresheet with one line per probe axis, and an axis you were *prompted* into scores lower than the same axis volunteered — because prompting is evidence that it was not part of your model until they said so.

The six prompts below are the canonical GPU-infra design surface. They recur, in near-identical shape, because they are the six things a fleet operator actually has to build. Drilling them is not memorising six architectures — there is rarely one correct architecture — it is making the *volunteering* reflexive, so that under pressure your attention goes to the interviewer's follow-up rather than to remembering what to say next.

## What's new here (calibration)

Nothing here is a new *system-design skill*. What is new:

- **The scoresheet model** — what an interviewer is actually writing down per axis, and why a mentioned axis and a defended axis are different scores.
- **A universal opening block** you can deliver on any of the six prompts in the first ninety seconds, written out to be memorised and then unlearned.
- **Six full drills**, each with the prompt as it is actually asked, a worked answer in speaking register, three follow-ups an interviewer will push with weak-versus-strong responses side by side, and the failure modes specific to that prompt.
- **The reversal-condition discipline** carried forward from lesson 02: every verdict you state must come with the condition under which you would change it. That single habit converts a preference into a decision.
- **Cross-referencing your own answers** — the synthesis credit that only appears when a P1 answer reaches into P2 and a P3 answer reaches into P6.

## Core concepts

### 1. What the round is actually scoring

A design round is not a search for the right architecture. It is a structured sample of how you think, scored against a small set of axes the interviewer knows in advance. Concretely, they are running something close to this per axis:

| Score | What it means | What it sounds like |
|---|---|---|
| **0** | never came up | the axis is absent from your design |
| **1** | mentioned | "we'd use MIG for isolation" |
| **2** | defended with a tradeoff | "MIG here, because attribution has to survive a chargeback dispute and only a hardware partition gives an exact split — I'd trade the stranded capacity for that. I'd reverse it if the tenant mix got bursty enough that static geometry stranded more than time-slicing wastes." |
| **−** | wrong, or hand-waved when pushed | "MIG basically makes GPUs multi-tenant" |

And separately, four *global* dimensions that are scored on whether they appeared **unprompted**:

- **Scale** — how many GPUs, what generation, how many tenants, what request or job rate.
- **Cost** — the dollar magnitude of the thing you are designing, with a basis and a date.
- **Failure modes** — what breaks, at what rate, and what the system does about it.
- **SLOs** — what you are actually protecting, stated as a measurable objective.

**The passing shape is: every axis ≥1, at least three axes at 2, and all four global dimensions volunteered in the first three minutes.** That is the target for every drill below.

```
   THE 45-MINUTE DESIGN ROUND, BLOCK BY BLOCK
  ══════════════════════════════════════════════════════════════════════════════

   00:00 ┌───────────────────────────────────────────────────────────────┐
         │ BLOCK A · FRAME  (3 min)                                      │
         │  · 2–3 clarifying questions, then STOP asking                 │
         │  · volunteer SCALE, COST, SLO, FAILURE MODES                  │
         │  · name the ONE dial this design turns                        │
         │ SCORED: all four global dimensions, unprompted                │
   03:00 ├───────────────────────────────────────────────────────────────┤
         │ BLOCK B · SPINE  (7 min)                                      │
         │  · the boxes and the data flow, end to end, no detail yet     │
         │  · say what each box is FOR, not what it is                   │
         │  · get to a complete-but-shallow system before going deep     │
         │ FAIL MODE: depth-first. You spend 20 minutes on the exporter  │
   10:00 ├───────────────────────────────────────────────────────────────┤
         │ BLOCK C · THE GPU LAYER  (12 min)  ◀── THE ROUND IS WON HERE  │
         │  · isolation model · attribution · scheduling constraint      │
         │  · the number that lies, and what you use instead             │
         │  · one worked calculation, out loud, with units               │
         │ SCORED: the axes at 2. This is where a generic answer dies.   │
   22:00 ├───────────────────────────────────────────────────────────────┤
         │ BLOCK D · PRESSURE  (15 min)                                  │
         │  · the interviewer pushes: "why not X?", "what if 10×?",      │
         │    "what breaks first?"                                       │
         │  · every answer: verdict + reason + REVERSAL CONDITION        │
         │ SCORED: whether your verdicts were reasoned or defaulted      │
   37:00 ├───────────────────────────────────────────────────────────────┤
         │ BLOCK E · CLOSE  (5 min)                                      │
         │  · what you'd build first, what you'd measure, what you cut   │
         │  · one honest limitation, volunteered                         │
   42:00 └───────────────────────────────────────────────────────────────┘

   THE TWO CLOCK ERRORS
     · Spending Block B's time in Block C  → an incomplete system at 22:00,
       and the interviewer never gets to push, so Block D never happens
     · Never leaving Block B               → a complete, generic, forgettable
       design. Scored: strong generalist.
```

### 2. The universal opening

Every one of the six prompts can be opened with the same ninety-second block. Memorise the *shape*, not the words, and let the numbers change.

> *"Before I draw anything, let me pin down the size of the problem, because at GPU prices the sizing decision dominates everything else.*
>
> *Assume a thousand accelerators of current generation, mixed training and inference tenants, on the order of ten teams. At two to three dollars per GPU-hour — on-demand basis, and I'd want to check that against your actual contract, because quotes in that band ranged from about two to eleven dollars depending on commitment — that is roughly twenty to twenty-five million dollars a year of capacity. So the SLO I am actually protecting here is [X], and the reason is that a one-percentage-point move in [X] is worth about [Y] a year.*
>
> *The failure modes I expect to dominate, in order: hardware failure at a rate that scales with job size — measured mean-time-to-failure for thousand-GPU jobs is on the order of hours, not days; silent under-utilisation, because the default metric measures kernel residency rather than work; and partial scheduling, where a distributed job gets most of its ranks and hangs on the rest while you bill for all of them.*
>
> *And the one dial this whole design turns is [isolation versus utilisation / accuracy versus explainability / latency versus throughput]. Everything below is me choosing where to sit on it. Does that framing match what you had in mind, or would you rather I optimise for something else?"*

**Why that last question matters.** It converts a monologue into a collaboration and gives the interviewer a cheap way to steer you toward the axes they care about. It also costs you nothing: if they say "that's right," you have their agreement on the frame for the rest of the round.

**The four global dimensions, in one block, in ninety seconds.** After that you never have to worry about them again, and the whole of your remaining attention goes on the GPU layer.

### 3. Drill P1 — Design a multi-tenant GPU platform

**The prompt, as asked:** *"We have a few hundred GPUs and a growing number of internal teams who all want them. Design the platform that lets them share it."*

**Probe axes:** isolation model · hard vs soft multi-tenancy · noisy-neighbour mechanisms · QoS, priority and preemption · fair-share quota · the security boundary · pooled vs dedicated node pools.

**The master dial, drawn.** Everything in P1 is a position on one spectrum, and being able to draw it is worth more than any single fact about MIG.

```
   THE ISOLATION ↔ UTILISATION DIAL   (and what each position costs you)
  ══════════════════════════════════════════════════════════════════════════════

   MORE ISOLATION ◀────────────────────────────────────────▶ MORE DENSITY

   ┌──────────────┬──────────────┬──────────────┬────────────────────────┐
   │ PASSTHROUGH  │     MIG      │     MPS      │     TIME-SLICING       │
   │ 1 pod : 1 GPU│ HW partition │ thread-% cap │ CUDA context switching │
   ├──────────────┼──────────────┼──────────────┼────────────────────────┤
   │ isolation:   │ isolation:   │ isolation:   │ isolation:             │
   │  total       │  hard — own  │  partial —   │  NONE. Shared address  │
   │              │  SMs, memory │  the daemon  │  space. NOT a security │
   │              │  paths, L2   │  caps thread │  boundary. Say this    │
   │              │              │  percentage  │  out loud.             │
   ├──────────────┼──────────────┼──────────────┼────────────────────────┤
   │ attribution: │ attribution: │ attribution: │ attribution:           │
   │  trivially   │  EXACT — an  │  BOUNDED     │  ESTIMATE ONLY. One    │
   │  exact       │  instance    │  estimate,   │  device counter, N     │
   │              │  belongs to  │  because the │  tenants, all reading  │
   │              │  one         │  cap bounds  │  the same value.       │
   │              │  container   │  the error   │                        │
   ├──────────────┼──────────────┼──────────────┼────────────────────────┤
   │ utilisation: │ utilisation: │ utilisation: │ utilisation:           │
   │  poor if the │  good, but   │  high        │  highest for bursty    │
   │  tenant      │  RIGID —     │              │  tenants               │
   │  can't fill  │  partitions  │              │                        │
   │  the card    │  don't       │              │                        │
   │              │  rebalance   │              │                        │
   ├──────────────┼──────────────┼──────────────┼────────────────────────┤
   │ stranding:   │ stranding:   │ stranding:   │ stranding:             │
   │  none        │  REAL — 7 ×  │  none        │  none                  │
   │              │  1g.10gb =   │              │                        │
   │              │  0.858 of an │              │                        │
   │              │  A100 on a   │              │                        │
   │              │  memory      │              │                        │
   │              │  basis       │              │                        │
   └──────────────┴──────────────┴──────────────┴────────────────────────┘

   AND THE SAME DIAL EXISTS ONE LEVEL UP, AT THE CLUSTER:
     dedicated node pools per tenant ◀──────▶ one pooled cluster with
       simple billing, poor utilisation        scheduler-enforced quota:
       small blast radius                      better utilisation, bigger
                                               blast radius, harder billing
```

**The worked answer** (this is the shape to speak, not to read):

> *"Frame first. Say three hundred GPUs, ten internal teams, mixed training and interactive notebooks, a couple of dollars per GPU-hour, so on the order of five to seven million a year. The SLO I'm protecting is **time-to-first-GPU for a queued job**, because in a shared cluster the thing teams actually complain about is waiting, and the failure mode that dominates is one team parking capacity they aren't using. The dial is isolation versus utilisation, and I'll turn it differently for different tenant classes.*
>
> *First question I'd resolve: is this hard or soft multi-tenancy? These are internal teams, cooperative, no adversarial trust boundary — so I do **not** need hardware isolation everywhere, and buying it everywhere would be expensive in stranded capacity. But I'd carve the tenant population into three classes rather than treating them uniformly.*
>
> *Class one, large training jobs: **passthrough**, whole GPUs, gang-scheduled. They can fill a card and they need the full memory. Nothing to share.*
>
> *Class two, production inference: **MIG**, because these are the tenants whose costs I will eventually charge back, and MIG is the only sharing mode where attribution is exact — a MIG instance belongs to exactly one container, so the split is a hardware fact rather than an estimate. I pay for that with stranding: seven `1g.10gb` slices only sum to about 86% of an A100 on a memory basis, so 14% of the card is physically unallocatable. I'd book that explicitly as platform overhead rather than hiding it, because the stranding is a geometry decision someone made and surfacing it is how the decision gets revisited.*
>
> *Class three, interactive notebooks: **time-slicing**, because they're bursty, mostly idle, and nobody is billing them. But I'd say the important thing out loud: time-slicing is not an isolation mechanism and not a security boundary. Those pods share an address space. If any of these tenants were external or untrusted, this class would not exist.*
>
> *Noisy neighbour is where GPU multi-tenancy differs from CPU multi-tenancy, so let me be specific. Even under MIG, memory bandwidth and the memory controller are shared paths on some access patterns; the L2 and the interconnect are shared. So the isolation MIG gives you is on SMs and framebuffer, not perfectly on bandwidth. Which means a bandwidth-hungry neighbour can still hurt a latency-sensitive tenant on the same card, and if I have a tenant with a real latency SLO I put them on a whole card rather than a slice. That is the sentence I'd want on the record.*
>
> *Quota: fair share with borrowing and reclaim, not fixed allocations. Fixed allocation is what produces the parking behaviour that is my dominant failure mode — a team holds what it was given because releasing means losing it. Kueue's model is the right shape: nominal quota per team, a cohort that lets idle quota be borrowed, and reclaim when the lender comes back. Preemption evicts the whole gang, never one pod, or you recreate the partial-placement hang.*
>
> *And I'd volunteer the cost measurement, because it's the part that makes the quota policy self-correcting: attribution per pod, on the honest counter, published as showback before anyone tries chargeback."*

**Follow-up 1 — "Why not just use MIG everywhere? It's the cleanest."**

| Weak | Strong |
|---|---|
| "MIG has some overhead and not all workloads fit the profiles." | "Three reasons, in order of how much they'd cost me. First, MIG partitions are static — reconfiguring geometry is a disruptive operation, so a fleet whose tenant mix shifts weekly spends its life either mis-partitioned or draining nodes to re-partition. Second, stranding: seven `1g.10gb` on an A100 sums to 0.858 of the card on a memory basis, so I'd give up about 14% of every partitioned card, which on three hundred GPUs at $2/hr is roughly $700k a year. Third, a large training job needs the whole card's memory and NVLink, so MIG would make class one impossible. I'd reverse this if the tenants became external or adversarial — then the security boundary argument outweighs all three, and I'd take the stranding." |

**Follow-up 2 — "A tenant says their job is slow and blames a neighbour. How do you settle it?"**

| Weak | Strong |
|---|---|
| "Check the metrics and see if the GPU is contended." | "I'd need to separate three hypotheses, and only one of them is the neighbour. First: is the tenant's own workload the problem? Look at their SM-active and tensor-active — a memory-bound job at batch 1 will be slow with no neighbour at all, and that is the most common answer. Second: is it bandwidth contention? `DRAM_ACTIVE` on the physical device, aggregated across all instances, tells me whether HBM is saturated; if the device is at 0.9 DRAM-active and the tenant's own tensor activity is low, they're queuing behind someone. Third: is it the fabric? If it's a distributed job, per-rank step-time outliers point at a rank, not at the GPU. The thing I would *not* do is settle it with the utilisation panel, because on a shared card that number reads high whenever anyone's kernel is resident, which tells me nothing about who." |

**Follow-up 3 — "Suppose one team is external. What changes?"**

| Weak | Strong |
|---|---|
| "We'd put them in their own namespace with network policy." | "The whole answer changes, because time-slicing stops being on the menu — it shares an address space, so it is not a security boundary regardless of what the namespace policy says. External tenants get passthrough or MIG, on dedicated nodes, and I'd move the cluster-level dial too: a separate node pool rather than a pooled cluster, accepting worse utilisation for a smaller blast radius. I'd also want to be honest that MIG's isolation guarantee is about SMs and framebuffer — the memory controller and interconnect are still shared paths, so a determined co-tenant has a side channel, and if the threat model includes that, the answer is whole nodes." |

**Failure modes on P1:** reciting the three sharing mechanisms as a fact-dump without landing on which you would pick *here*; treating the isolation dial as existing only at the GPU level and missing the cluster-level version; calling time-slicing an isolation mechanism; describing quota without naming borrowing and reclaim, which is what actually fixes parking; never mentioning cost, which is the whole reason multi-tenancy exists.

### 4. Drill P2 — GPU cost attribution, showback and chargeback

**The prompt:** *"Finance wants to know what each team spent on GPUs last month. Design that."*

**This is your home-field prompt.** Volunteer it if given any choice of prompt, and open by naming the attribution formula *and its flaws* before being asked. Probe axes: telemetry source · the attribution formula and its failure modes · allocated versus utilised ledgers · shared-endpoint splitting · showback versus chargeback · schema normalisation · dispute handling.

**The worked answer:**

> *"Frame: a thousand GPUs, ten teams, two to three dollars per GPU-hour on-demand — call it twenty to twenty-five million a year, which is why this question exists. The SLO here is unusual and worth stating: it is **explainability**, not accuracy. A number that is 5% off but that a team lead can reconstruct beats a number that is exact and opaque, because the failure mode of a chargeback system is not arithmetic error, it is a team disputing a bill, losing confidence, and the programme being switched off.*
>
> *Start with what the number can even be. There are two ledgers and they answer different questions. **Allocated** GPU-hours come from the scheduler: which pod held which device for how long. That's a fact, it's exact, and it's what the invoice looks like. **Utilised** GPU-hours come from hardware counters: how much of that time did the SMs actually have work. That's a measurement, it's honest, and it's the efficiency signal. I emit both and I never blend them, because the moment you blend them you cannot tell a scheduling complaint from an efficiency complaint.*
>
> *Telemetry: DCGM exporter as a DaemonSet, with the Kubernetes flag on so it joins pod identity from the kubelet's pod-resources API. The two data paths meet at exactly one key — the physical GPU UUID — and everything downstream depends on that join being right, which is why I'd make it testable.*
>
> *Now the formula everybody writes: team cost equals team utilisation over total utilisation, times the GPU-hour cost. I'd flag three flaws before you ask. One, utilisation is not value — a low-utilisation inference endpoint can be revenue-critical, and billing it as waste is politically fatal. Two, reserved-but-idle: a team holds a card at 5%; do you bill the reservation or the usage? Three, and this is the one people miss, that formula literally rewards busy-looping. A team that spins a kernel to look busy pays less per unit of real work. So I would not bill on utilisation at all — I bill on **allocation**, which is a scheduler fact with no measurement exposure, and I publish utilisation as the efficiency lens next to it.*
>
> *Which metric for the efficiency lens matters. Not the standard utilisation field — that reports whether a kernel was resident, not whether work happened; a batch-1 decode server pins it at 99 while 16% of the SMs are lit. I use SM-active for the waste claim and tensor-active for the efficiency claim, and I say which is which.*
>
> *The hard part is sharing. Under MIG it's clean: an instance belongs to one container, and I cost a `1g.10gb` as one seventh of the card on an SM-slice basis, with the basis emitted as a label because compute-share and memory-share disagree by about 16% relative for the same slice. Under time-slicing it is **not** clean and I won't pretend otherwise: the counter is device-scoped, not context-scoped, so N tenants read one number. I emit a request-weighted approximation plus an explicit confidence flag, and I quantify how much of the total rests on it — on a fleet where 24 of 64 GPUs are time-sliced, that was about 47% of utilisation-based dollars, roughly $64k a month. Naming that number is what makes the rest of the system credible.*
>
> *Roll-out: showback first, for at least a quarter, with a dispute process and a snapshot ID on every row so any disputed number can be replayed. Chargeback only after the numbers survive disputes. And the schema is FOCUS-shaped so the on-prem and cloud halves are comparable — the split-cost-allocation columns that landed in FOCUS 1.3 carry the per-tenant ratios, and I add exactly two extensions, because the standard still has no utilisation ledger and no confidence band on an estimated split."*

**Follow-up 1 — "Doesn't OpenCost already do this?"**

| Weak | Strong |
|---|---|
| "It doesn't handle GPUs very well." | "It does the allocated ledger, correctly, for exclusively held whole GPUs. Where it stops is the numerator: GPU cost is `GPUHours × CostPerGPUHr`, where `GPUHours` comes from the pod's whole-device resource *request* count and the price is one scalar per physical device. That expression has exactly two factors, an integer and a per-device price, and a sub-device fraction is neither — so it needs a third term, which is an input-model change rather than a setting. Concretely: on the same H100 at the same price, seven MIG tenants each bill a whole card, so the card bills 7×; and under time-slicing with the feature-discovery rename on, the shared resource key is absent from both numerator paths and the pods bill **zero** while the platform absorbs a fully-used card. Same tool, same hardware, four different answers depending on a GPU-operator flag. Two of those gaps are a few lines and worth upstreaming; the fractional numerator is not." |

**Follow-up 2 — "A team disputes their bill. Walk me through what happens."**

| Weak | Strong |
|---|---|
| "We'd look at the metrics together and figure it out." | "First, they can only dispute what I can replay, so every row carries the snapshot key of the pod-resources state it was derived from — I can reconstruct exactly which devices their pods held, minute by minute. Second, I'd expect the dispute to be one of three kinds and each has a different answer. If they say 'we didn't use it' — correct, and irrelevant, because the bill is on allocation and I'd show them the utilised series next to it as the actionable number. If they say 'that wasn't our pod' — that's a join bug and I'd check the conservation identity for that device; the shares plus unallocated must sum to exactly 1.000, and a violation localises it. If they say 'we shared that card' — then I'd show them the confidence label on those rows and tell them honestly it's an estimate, and how big the estimated portion of their bill is. The thing that makes this survivable is that I published the estimate fraction before they asked." |

**Follow-up 3 — "How do you split a shared inference endpoint that serves five teams?"**

| Weak | Strong |
|---|---|
| "Split by request count." | "Request count is the wrong unit because requests are not the same size — one team's 4,000-token generation costs far more than another's 50-token classification. I'd split by compute time, which for a serving stack means using the prefill and decode timing counters: input cost proportional to prefill time, output cost proportional to decode time, so a token-heavy tenant pays for their tokens. If those counters aren't available, the fallback is a fixed output-to-input multiplier — and I'd record which method was used per row, because a fallback silently applied is how a cost model stops being auditable. Above that layer, the endpoint's own GPUs are attributed the same way as anything else; the token split only distributes the endpoint's cost among its tenants." |

**Failure modes on P2:** billing on utilisation without naming that it rewards busy-looping; presenting one blended number instead of two ledgers; claiming per-tenant accuracy under time-slicing; using the standard utilisation field as the efficiency metric; skipping the showback-first stage and going straight to billing; never quantifying how much of the total is an estimate.

### 5. Drill P3 — Design a training-job scheduler

**The prompt:** *"Teams submit distributed training jobs of 8 to 512 GPUs. Design the scheduler."*

**Probe axes:** gang / all-or-nothing scheduling · quota borrowing and reclaim · preemption granularity · fair share with ageing · topology-aware placement · scheduler choice · what happens on failure mid-job.

**The failure this prompt is really about, drawn as a causal chain:**

```
   WHY VANILLA KUBERNETES SILENTLY BURNS MONEY ON A DISTRIBUTED JOB
  ══════════════════════════════════════════════════════════════════════════════

   user submits a 4-rank job (4 pods, 1 GPU each)
              │
              ▼
   kube-scheduler binds pods INDEPENDENTLY — it has no concept of a gang
              │
        ┌─────┴──────────────────────────────┐
        ▼                                    ▼
   3 pods find a GPU               1 pod finds none → Pending
   and START                                 │
        │                                    │
        ▼                                    │
   ranks 0,1,2 initialise NCCL               │
   and enter the first collective            │
        │                                    │
        ▼                                    │
   ┌──────────────────────────────┐          │
   │ ALL-REDUCE BLOCKS waiting    │◀─────────┘  rank 3 never arrives
   │ for rank 3                   │
   └──────────────┬───────────────┘
                  │
     ┌────────────┴─────────────────────────────────────────┐
     ▼                                                      ▼
   NO ERROR is raised. The job is "Running".      THE WAIT IS IMPLEMENTED
   Logs are silent. Dashboards look fine.         AS A RESIDENT KERNEL,
     │                                            so GPU_UTIL reads ~100
     ▼                                            on all three idle ranks
   YOU ARE BILLED FOR 3 IDLE GPUs, INDEFINITELY        │
     │                                                  ▼
     └──────────────────────────────────────▶  the monitoring you have
                                                CANNOT SEE THIS. The only
                                                signal is that tensor
                                                activity collapsed while
                                                utilisation stayed pinned.

   FIX: all-or-nothing (gang) admission — either every rank is placed
        or none is. And preemption must evict THE WHOLE GANG, or you
        have just recreated the same state from the other direction.
```

**The worked answer:**

> *"Frame: jobs from 8 to 512 GPUs, so the largest job is a meaningful fraction of the fleet; say a thousand GPUs total, two to three dollars an hour. The SLO is **queue wait for a job that fits**, and the failure modes that dominate are partial placement, fragmentation, and hardware failure mid-run — because at this scale, measured mean-time-to-failure for thousand-GPU jobs is on the order of hours, so a 512-GPU job that cannot survive a node loss will not finish.*
>
> *The headline requirement is **gang scheduling**, and I'd explain why in cost terms rather than mechanism terms: Kubernetes binds pods independently, so a four-rank job can get three ranks running and one Pending. The three that started enter the first collective and block. Nothing errors. The job says Running. And — this is the part that makes it expensive — the collective wait is implemented as a resident kernel, so those three GPUs report near-100% utilisation while doing nothing. You are billed for three idle GPUs and your dashboard says they're busy. All-or-nothing admission is the fix, and preemption has to evict the whole gang for the same reason.*
>
> *Quota: nominal quota per team, grouped into a cohort so idle quota can be borrowed, with reclaim when the lender returns. That's the mechanism name — cohorts — and it matters because the alternative, fixed per-team allocations, produces hoarding. Add ageing so a long-waiting job's effective priority rises, or large jobs starve behind a stream of small ones. That starvation is the specific pathology of a fair-share scheduler on a GPU fleet: a 512-GPU job needs 512 free slots simultaneously, and a steady drizzle of 8-GPU jobs means that moment never arrives. The fix is reservation — accumulate and hold nodes for the big job rather than backfilling forever.*
>
> *Topology is a first-class scheduling constraint here, not an optimisation. Ranks co-located within one NVLink domain communicate at intra-node bandwidth; ranks split across nodes drop to network bandwidth, and ranks split across network rails drop again. So placement is not 'find 512 free GPUs', it's 'find 64 whole nodes with the right fabric adjacency'. I'd want the scheduler to understand node groupings and to prefer whole-node, whole-rail allocation even at the cost of longer queue wait — and I'd make that tradeoff explicit rather than silent, because a job that starts sooner and runs 40% slower is worse for everyone.*
>
> *On the scheduler choice: Kueue for the quota and borrowing layer on top of the Kubernetes scheduler; Volcano or the open-sourced KAI scheduler if I need gang plus hierarchical queues in one component; Slurm if the tenant population is HPC-native and already writes batch scripts. And I'd name **Dynamic Resource Allocation**, which graduated to GA in Kubernetes 1.34 and is enabled by default from that release, as the direction device allocation is moving — it gives structured, claim-based allocation instead of an opaque integer resource count, which is directly relevant here because it also makes attribution cleaner.*
>
> *Failure mid-job: the scheduler has to cooperate with checkpointing and with health signals. A node that fails takes the gang down; the job restarts from the last checkpoint; so the scheduler needs to re-place the whole gang, and the health system needs to have already cordoned the bad node or you re-place onto it."*

**Follow-up 1 — "Gang scheduling reduces utilisation. Justify it."**

| Weak | Strong |
|---|---|
| "It's necessary for distributed jobs to work." | "It does, and the tradeoff is real: holding partial allocations while waiting for a full gang leaves GPUs idle that a greedy scheduler would have filled. But compare the two wastes. Gang scheduling's waste is *bounded and visible* — it's queue-wait time on nodes you can measure and cap. Partial placement's waste is *unbounded and invisible* — the job never completes, the GPUs report full utilisation, and nothing alerts. I'd take bounded and visible every time. I'd also reduce the gang-scheduling cost rather than abandon it: backfill small jobs into the reservation window, and cap how long a partial reservation may be held before it's released." |

**Follow-up 2 — "How do you stop a 512-GPU job from starving?"**

| Weak | Strong |
|---|---|
| "Give it higher priority." | "Priority alone doesn't fix it, because the problem isn't ranking, it's simultaneity — the big job needs 512 slots at one instant, and with continuous small-job churn that instant never occurs. So: reservation with ageing. As the big job waits, its effective priority rises, and once it wins, the scheduler *accumulates and holds* freed nodes for it rather than backfilling them, with a deadline so a reservation can't be held forever. Small jobs can still backfill into the reservation if they'll finish before the reservation completes — that's the classic HPC backfill and it recovers most of the lost utilisation. The number I'd watch is reservation drain time, because if that grows the fleet is too fragmented and the real fix is upstream, in how the small jobs are packed." |

**Follow-up 3 — "Does topology-aware placement really matter at 8 GPUs?"**

| Weak | Strong |
|---|---|
| "Yes, NVLink is faster than Ethernet." | "At 8 GPUs it matters *more* per rank, not less, because 8 GPUs is exactly one node on most current hardware — so this is a binary: either the job fits inside one NVLink domain or it doesn't, and the collective bandwidth differs by roughly an order of magnitude between those two cases. I'd turn it into a number rather than an assertion: take the model's gradient size, divide by the collective's effective bandwidth in each placement, and that's the per-step communication time; compare it to the compute time per step. If communication becomes a meaningful fraction of step time, placement is a first-class constraint and I'd rather wait for a whole node. If the model is small enough that communication is a few percent, I'd take the earlier start. The point is that it's an arithmetic question, and I'd want the scheduler's policy to be derived from that arithmetic rather than from a preference." |

**Failure modes on P3:** describing gang scheduling mechanically without stating the cost consequence; saying "quota borrowing" without naming the cohort mechanism; treating topology as an optimisation rather than a constraint; ignoring the interaction with failure and checkpointing; presenting one scheduler as the only option.

### 6. Drill P4 — Design a model-serving platform

**The prompt:** *"Design a platform that serves many LLMs to many tenants with good latency and good cost."*

**Probe axes:** prefill versus decode · KV-cache management · continuous batching · TTFT and inter-token latency SLOs · multi-model packing and cold start · autoscaling and scale-to-zero · quantisation.

**This is the prompt generic candidates hand-wave**, so the GPU layer is where all the credit is.

```
   WHY SERVING HAS TWO DIFFERENT BOTTLENECKS IN ONE REQUEST
  ══════════════════════════════════════════════════════════════════════════════

   ONE REQUEST'S LIFE                                  what limits it
   ─────────────────────                               ──────────────

   t0  request arrives
       │
       ▼
   ┌────────────────────────────────────────────┐
   │ PREFILL — process the whole prompt at once │   COMPUTE-BOUND
   │  · one big matrix-matrix multiply per layer│   arithmetic intensity is
   │  · all prompt tokens in parallel           │   high; tensor pipes are
   │  · cost ∝ prompt length                    │   genuinely busy
   └───────────────────┬────────────────────────┘   → sets TTFT
                       │  KV cache for the prompt is now resident
                       ▼
   ┌────────────────────────────────────────────┐
   │ DECODE — one token at a time, N times      │   MEMORY-BANDWIDTH-BOUND
   │  · matrix-VECTOR per layer per token       │   every weight is re-read
   │  · re-reads ALL weights each step          │   from HBM per step
   │  · KV cache GROWS each step                │   → arithmetic intensity
   │  · cost ∝ output length                    │     ≈ 1 FLOP/byte at batch 1
   └───────────────────┬────────────────────────┘     vs a ridge point of
                       │                              ~295 on an H100
                       ▼                            → sets inter-token latency
   t1  last token emitted; KV cache freed

   CONSEQUENCES YOU MUST SAY OUT LOUD
     1. Batching is the ONLY lever that moves decode, because it amortises
        the weight read across B sequences. Arithmetic intensity ≈ B.
     2. The binding capacity constraint is KV-CACHE MEMORY, not FLOPs.
        Concurrency is limited by how many sequences' caches fit in HBM.
     3. Prefill and decode want different batch policies, which is why
        continuous (iteration-level) batching beats static batching:
        requests join and leave the running batch every decode step.
     4. A decode-heavy fleet pins the utilisation duty cycle at ~100
        while the tensor pipes run near 1%. The dashboard cannot see this.
```

**The worked answer:**

> *"Frame: say a hundred models, a few dozen tenants, traffic that's bursty and long-tailed, on the order of a hundred GPUs, two to three dollars an hour. Two SLOs, and naming both is important because they trade against each other: **time to first token**, which is a prefill latency, and **inter-token latency**, which is a decode smoothness property. Throughput trades against both, and the dial this design turns is latency versus throughput via batch size.*
>
> *The single most important thing to establish is that a request has two phases with two different bottlenecks. Prefill processes the whole prompt at once — matrix-matrix multiplies, high arithmetic intensity, genuinely compute-bound, and it sets TTFT. Decode generates one token at a time, re-reading every weight from HBM at each step — matrix-vector, memory-bandwidth-bound, and it sets inter-token latency. For an 8B model in BF16 on an H100, one decode step at batch 1 moves 16 GB of weights at 3.35 TB/s, which is about 4.8 milliseconds, wrapped around roughly 16 microseconds of arithmetic. That's the whole reason batching is the lever: batching amortises the weight read across B sequences, so arithmetic intensity climbs from about 1 FLOP per byte toward the ridge point around 295.*
>
> *So: **continuous batching**, not static. Requests join and leave the running batch at every decode step, rather than waiting for a batch to fill and then for its slowest member to finish. That is the single biggest throughput lever available and it costs a little inter-token jitter.*
>
> *The capacity constraint is **KV cache, not FLOPs**, and I'd say that explicitly because it's the thing that determines how many concurrent sequences you can hold. Each active sequence's cache grows with its length; when HBM fills, you either reject, evict or swap. Paged block allocation is the mechanism that makes this tractable — fixed-size blocks rather than contiguous per-sequence arenas, so fragmentation stops being what limits concurrency. Capacity planning for this platform is a memory calculation, not a FLOPs calculation.*
>
> *Multi-model packing: a hundred models will not fit resident. So there's a hot/warm/cold tier — the top handful of models resident on dedicated capacity, a middle tier sharing GPUs, and a long tail loaded on demand. That makes cold start the dominant tail-latency term, and the cold-start cost is dominated by getting weights into HBM, so the levers are weight locality, a faster load path, and keeping some capacity warm. Scale-to-zero is a straight tradeoff: idle GPU cost against cold-start latency, and it is only correct for tenants whose latency SLO tolerates it.*
>
> *Quantisation is a cost lever with a quality cost — FP8 or INT8 roughly halves or quarters the bytes moved per token, which directly attacks the decode bottleneck, at some accuracy cost that has to be measured per model rather than assumed.*
>
> *And the observability layer, because it's the part that bites: a decode-heavy fleet pins the standard utilisation metric near 100 while the tensor pipes run around 1%. If your autoscaler scales on utilisation, it will never scale anything, because the metric is already saturated. Scale on queue depth and on the SLO — time to first token and inter-token latency — not on device utilisation."*

**Follow-up 1 — "How many concurrent requests can one H100 hold?"**

| Weak | Strong |
|---|---|
| "Depends on the model — maybe a few dozen." | "It's a memory calculation and I'd do it out loud. Start with 80 GB of HBM. Subtract the weights: an 8B model in BF16 is 16 GB. Subtract framework and activation overhead — call it a few gigabytes. What's left, roughly 60 GB, is the KV-cache arena. Per-sequence cache size is `2 × layers × kv_heads × head_dim × bytes_per_element × sequence_length`, and the factor of 2 is keys plus values. So concurrency is that 60 GB divided by the per-sequence cache at your expected context length — which means concurrency falls roughly linearly as context length grows, and a long-context tenant costs many times a short-context one for the same request count. That is also the honest answer to why a per-request cost model is wrong and a per-token one is right." |

**Follow-up 2 — "Your P99 time-to-first-token regressed. Where do you look?"**

| Weak | Strong |
|---|---|
| "Check GPU utilisation and scale up." | "TTFT is prefill plus queueing, so I'd split those first — if queue time grew, it's admission and capacity; if prefill time grew, it's the workload or the hardware. Most TTFT regressions are queueing, and the usual cause under continuous batching is that long decodes are occupying the batch, so new prompts wait. That points at a scheduling policy question — whether prefill is allowed to preempt or interleave with decode — rather than at capacity. If it's genuinely prefill time, I'd look at prompt-length distribution shifting, at cache-hit rate on shared prefixes, and only then at the hardware. And I would specifically not scale on device utilisation, because a decode-heavy fleet reads pinned regardless." |

**Follow-up 3 — "Scale to zero on all hundred models?"**

| Weak | Strong |
|---|---|
| "Yes, it saves a lot of money on idle GPUs." | "Only for the tail. It's a straight exchange of idle GPU-hours for cold-start latency, so the right answer is per-tenant and driven by their SLO. For a tenant whose SLO tolerates seconds, scale to zero — the idle cost is real money and the model is cold most of the day. For an interactive tenant, never — the first request after an idle period pays the whole weight-load penalty and that is exactly the request a user is watching. In between, keep one warm replica and scale the rest. I'd also point out that scale-to-zero interacts badly with a naive reclaim policy: a warm-but-idle replica has near-zero SM activity and tens of gigabytes resident, so any reclaim rule that looks only at activity will kill exactly the replicas you're paying to keep warm. Gate on framebuffer." |

**Failure modes on P4:** staying at the platform layer and never descending into prefill/decode; describing paged attention correctly but still answering capacity questions in FLOPs; naming one SLO instead of two; proposing utilisation-based autoscaling; treating quantisation as free.

### 7. Drill P5 — Health detection and automated remediation

**The prompt:** *"Jobs on our fleet fail more often than they should and nobody knows why. Design detection and remediation."*

**Probe axes:** failure taxonomy · lemon-node detection · fleet-scale diagnostics scheduling · straggler detection · tiered remediation · cordon/drain interaction with the gang scheduler · the passes-idle-fails-under-load trap.

**The worked answer:**

> *"Frame: a thousand GPUs, jobs from 8 to 512 ranks. The thing that makes this a hard design rather than a monitoring exercise is the failure-rate arithmetic: measured mean-time-to-failure for 1,024-GPU jobs is about 7.9 hours in the published study of two large research clusters, and the same work projects shorter times at larger scale. At that cadence, detection latency is a direct cost lever — every minute a job runs on a sick node is billed silicon.*
>
> *The design starts with a taxonomy, because different failure classes need different responses. **Immediate** failures — Xid errors, a GPU falling off the bus, uncorrectable ECC — are loud, and the response is automated: cordon, drain, reschedule. **Degraded** nodes — thermal throttling, a clock that never reaches boost, a link that renegotiated to a lower width — are quiet and expensive, and the response is detection by comparison against fleet peers rather than against a threshold. **Intermittent** faults are the worst class, and they are the reason this design exists.*
>
> *The intermittent class has a name: **lemon nodes**. A lemon node passes every health check and repeatedly kills large jobs. The published study of those two clusters found that proactive lemon-node detection dropped the failure rate of jobs at 512 GPUs and above from about 14% to about 4%, identifying on the order of forty faulty nodes at over 85% detection accuracy. That's the single strongest argument for building this: it's a measured, published, order-of-magnitude effect on large-job success.*
>
> *Detecting them requires a different signal, because by construction they pass point-in-time checks. The signal is **association**: for each node, the failure rate of jobs that touched it, compared against the fleet baseline, over a window. That's a statistical claim rather than a threshold, which means you need enough job-touches per node for significance, and you need to handle the confound that big jobs touch many nodes and fail more often anyway.*
>
> *Diagnostics: DCGM's diagnostic levels give you the escalation ladder, and the key operational fact is that the load-level diagnostic is a stress test — you cannot run it on a node that's serving a job. So it goes in maintenance windows, on drained nodes, and its coverage is a capacity decision: running it nightly across the fleet costs real GPU-hours, and that cost has to be compared against the failure rate it prevents.*
>
> *Stragglers are a separate detection problem that looks like a health problem. One slow rank throttles an entire collective — the other ranks block, and because a collective wait is implemented as a resident kernel, they all report high utilisation while doing nothing. The signal is per-rank step-time outliers, not device metrics. I'd surface a straggler as a first-class event with the rank identified, because 'the job is slow' is unactionable and 'rank 47 on node-19 is 30% slower per step' is a cordon decision.*
>
> *Remediation is tiered: soft — reset the GPU, restart the agent; medium — cordon, drain, reschedule; hard — remove from the schedulable pool and open an RMA. The automation boundary belongs between medium and hard: automating an RMA decision is expensive to get wrong and rare enough that a human should look. And cordon/drain has to cooperate with the gang scheduler, because draining one node of a 512-rank job means re-placing the whole gang, not one pod.*
>
> *Finally the trap I'd name unprompted: **passes idle, fails under load.** Idle health checks are necessary and insufficient. Thermal, power and link-width faults only appear under sustained load, which is exactly when you cannot run the diagnostic. That gap is why the association-based lemon detection exists — it is the only signal that works on a node in production."*

**Follow-up 1 — "Your lemon detector has false positives. What does that cost?"**

| Weak | Strong |
|---|---|
| "We'd tune the threshold." | "It costs capacity and it costs trust, and those have different remedies. Capacity: each false positive removes a healthy node from the pool, so on a fleet where I'm cordoning even a few percent spuriously, I'm paying for GPUs I refuse to schedule — quantifiable directly as cordoned GPU-hours times the rate. Trust is worse, because if operators believe the detector is noisy, they'll start overriding it, and then the true positives get overridden too. So I'd make the detector's output *evidence-bearing* rather than binary: not 'node-19 is bad' but 'node-19 was involved in 6 of the last 9 large-job failures, versus a fleet baseline of 1.2 — here are the job IDs.' A human overriding that has to argue with the data. And I'd send suspected nodes to a drain-and-diagnose queue rather than straight to RMA, so a false positive costs one diagnostic window instead of a node." |

**Follow-up 2 — "A job is running at half speed. Is that a health problem?"**

| Weak | Strong |
|---|---|
| "Probably a slow node — check the metrics." | "It's three hypotheses and I'd bisect rather than guess. First, is it the same on every rank? If all ranks are uniformly slower, it's not a node — it's the workload, the data pipeline, or a configuration change. Second, if one rank is slower, that's a straggler and I'd look at that node's clocks and link width, because thermal throttling and a renegotiated PCIe link are the two classic causes of a node that's healthy but slow. Third, if no rank is slower but the collective takes longer, it's the fabric, not the GPUs. The metric discipline that makes this work: I do not use the device utilisation number for any of it, because on a job blocked in a collective, every waiting rank reports near-100% while doing nothing. The pair that separates them is SM-active staying high while tensor activity collapses — that shape is a stall, not work." |

**Follow-up 3 — "How often do you run the load diagnostic across the fleet?"**

| Weak | Strong |
|---|---|
| "Nightly." | "It's an explicit capacity tradeoff and I'd size it rather than assert a cadence. The diagnostic takes a node out of service for its duration, so fleet-wide nightly coverage costs `nodes × duration` GPU-hours per night — on a 125-node fleet with a fifteen-minute level-3 run, that's about 31 node-hours a night, or roughly 1.3% of capacity. Against that, weigh the failure rate it prevents: if lemon-node detection takes large-job failure from 14% to 4%, and a 512-GPU job failure costs its elapsed time since the last checkpoint, the arithmetic usually favours running it. But I'd stage it: full coverage on nodes returning from maintenance and on nodes flagged by the association detector, sampled coverage on the rest, and I'd tune the sample rate against the observed lemon-detection rate rather than a calendar." |

**Failure modes on P5:** treating health as a threshold problem when the expensive class is statistical; proposing idle health checks without naming the passes-idle-fails-under-load gap; automating the RMA decision; forgetting the gang-scheduler interaction on drain; using device utilisation to diagnose a stalled collective.

### 8. Drill P6 — Design the fleet's observability

**The prompt:** *"Design observability for a GPU fleet. What goes on the dashboard and what pages someone?"*

**Probe axes:** the metric hierarchy · why the default metric lies · the collection pipeline · cardinality budget · what to alert on and what never to alert on · absence as a failure signal · comms-versus-compute attribution.

**The worked answer:**

> *"Frame: a thousand GPUs, ten tenants, twenty-plus million a year of capacity. The SLO I'd protect is **goodput** — useful work per GPU-hour — and the reason to say that first is that it immediately rules out the metric everyone starts with.*
>
> *The hierarchy, from least to most useful. Device presence — is a kernel resident. Device breadth — what fraction of SMs have work. Device depth — how full the SMs are. Pipe activity — are the expensive units firing. And at the top, application throughput — tokens or samples per second, and model-FLOPs utilisation if the tenant can give you FLOPs per token. Each rung is closer to business value and further from something the platform can measure alone.*
>
> *The rung everyone stops at is the first one, and it does not mean what people think. The standard utilisation field is an unmodified passthrough of a driver counter defined as the percentage of a short sample window during which at least one kernel was resident. It's a threshold at one — one kernel and ten thousand kernels give the same answer, and it has no notion of how many SMs exist. On a batch-1 decode server it reads 99 while 16% of the SMs are lit and the tensor pipes are at 1%. So it goes on exactly one panel, directly beside the honest metric, as the foil that makes the gap visible.*
>
> *The pipeline is DCGM exporter as a DaemonSet, Prometheus scraping it, joined to pod identity through the kubelet's pod-resources API. Two configuration facts I'd flag because both are defaults that silently break things. One: the SM-breadth field ships **commented out** in the exporter's default counter set and needs elevated privileges — so out of the box you get two presence metrics and no breadth metric. Two: the exporter's pod labels collide with the labels Prometheus attaches from service discovery, and with the default collision policy the scraped ones get an `exported_` prefix — so a dashboard grouped by namespace attributes the whole cluster's GPU-hours to the operator's namespace, and looks entirely plausible while doing it. I'd normalise both in one recording rule and write everything downstream against that.*
>
> *Cardinality is a real budget, not a footnote. Labelling by tenant times GPU times model is a cross product and it explodes. I'd size it: roughly ten labels and a handful of series per GPU is on the order of forty series per GPU, so eight thousand GPUs is a few hundred thousand series — which is a Prometheus sizing input. The mitigations are recording rules that pre-aggregate the panels you actually use, a hard cap on per-container labels, and keeping per-process series out of the export entirely.*
>
> *Alerting policy, and this is the part I'd state as rules. Page on SM-breadth being near zero for a sustained window, **gated on framebuffer** — because a paused-but-loaded serving replica has near-zero activity and tens of gigabytes resident, and reclaiming it destroys a warm replica and gets the whole reclaim system switched off. Warn, never page, on tensor activity: efficiency is a conversation, not an incident. Never alert on the presence metrics at all, because a reclaim rule built on them will never fire on your most expensive waste — that waste is precisely the case that keeps a kernel resident.*
>
> *And the rule nobody writes: **alert on absence.** The profiling fields are dropped rather than zeroed when they're unavailable, and absence is not zero in PromQL — an average silently skips the missing GPU, so a fleet where half the cards stopped reporting the honest metric shows an unchanged, healthy average. A simple `unless` between a device field and a profiling field catches it. That's the single most common way an 'everything looks fine' GPU dashboard lies by omission."*

**Follow-up 1 — "Why not just alert when utilisation drops below 50%?"**

| Weak | Strong |
|---|---|
| "Because utilisation isn't accurate." | "Because that alert has a systematic blind spot exactly where the money is. The metric is a presence duty cycle, so it reads near 100 on every one of the expensive waste cases: batch-1 decode, spin-wait kernels, a collective blocked on a straggler, a dataloader-starved training loop. Every one of those keeps a kernel resident. So a threshold alert on that field will fire on genuinely idle cards — which are the cheap, obvious case you'd have noticed anyway — and stay silent on a fleet burning money at full rate. It's not that the number is inaccurate; it's accurately reporting a fact that is uncorrelated with the thing you want to know." |

**Follow-up 2 — "Your Prometheus is falling over. What do you cut?"**

| Weak | Strong |
|---|---|
| "Reduce the scrape interval and drop some labels." | "I'd cut by asking what each series is *for*, in three tiers. Tier one, the series that alerts depend on — SM-breadth, framebuffer, health codes, and the device field needed for the absence check. Those stay at full resolution. Tier two, the series that dashboards depend on — I'd replace the raw series with recording rules at the aggregation the panels actually use and drop the raw ones after a short retention, because nobody queries per-GPU tensor activity from six weeks ago. Tier three, diagnostic series — per-sub-pipe breakdowns, media engines, per-link counters — collected only on demand or on a sampled subset of nodes. And I'd check the integration constant before touching the scrape interval, because the GPU-hours arithmetic depends on sample spacing and changing it silently changes every cost figure downstream." |

**Follow-up 3 — "How do you know your dashboard is telling the truth?"**

| Weak | Strong |
|---|---|
| "We validate it against the workloads." | "Four checks, and I'd ship them as panels rather than run them once. Reconciliation: the buckets I decompose the fleet into must sum to the physical GPU-hours DCGM can see, within about 1%. Ordering: utilised must never exceed allocated — that query must return nothing, and any result means work is running on a GPU with no allocation record. Completeness: at 30-second sampling a day should contain 2,880 samples per series, and anything materially below is a gap that makes waste look smaller. And the one that actually proves the arithmetic — a synthetic ground-truth run: saturate one GPU for exactly 600 seconds, expect 0.1667 GPU-hours, accept about 0.150 to 0.184 for the quantisation error. If it returns 0.30 I'm double-counting; if it returns 4.0 I've extrapolated a mean over a window instead of integrating." |

**Failure modes on P6:** answering with "more dashboards"; proposing per-tenant labelling without naming the cardinality cost in the same breath; alerting on the presence metric; missing the absence case; not knowing that the honest field is off by default.

### 9. The drill protocol

Run each prompt as a **thirty-five-minute rep**: three minutes to clarify and deliver the universal opening, twenty-five minutes designing out loud and recorded, seven minutes self-scoring against that prompt's axes. Score each axis 0/1/2 as in §1. **Target: every axis ≥1, at least three axes at 2, and all four global dimensions volunteered inside the first three minutes.**

Log the scores. Re-drill any prompt under 70% until you get two clean consecutive reps. Two full passes over all six prompts is the week's work.

**Record yourself and listen back once.** Not for content — for two specific things: how long you spent before saying a number, and whether any verdict came out without its reversal condition. Both are audible and neither is visible from a transcript.

## Perspectives

**The interviewer's scoresheet view.** They are not looking for the right architecture; they are ticking axes and grading each 0/1/2. Volunteering is not a soft skill, it is literally the difference between a 1 and a 2, because an axis you were prompted into is evidence it was not in your model. The other thing they are watching for is *order*: reciting the checklist in a fixed sequence reads as rehearsed, whereas hitting the same points because the design genuinely needs them, in whatever order the conversation goes, reads as reasoning. Which is why these drills are structured as answers with follow-ups rather than as lists to memorise.

**The finance view on P2.** Picture a finance partner rather than an engineer reading your attribution formula. They do not care that it is elegant; they care whether it survives a dispute. A team that reserved a card and used 5% of it will contest a usage-based bill. A team running a low-utilisation but revenue-critical endpoint will contest being billed as if idle time were waste. The formula's flaws are not academic footnotes; they are the exact objections raised in the first review meeting, which is why volunteering them unprompted is the tell of someone who has sat in that meeting.

**The on-call view on P3 and P5.** Gang-scheduling failures and lemon nodes do not appear as diagrams at 03:00; they appear as a page saying a 512-GPU job has been at 60% progress for six hours, or that a run dies whenever it touches one rack. The engineer holding the pager does not care about the elegance of your cohort hierarchy; they care whether the design gives them a fast, cheap signal — a per-rank step-time outlier, a repeat-offender node log — that turns "mysteriously stuck" into "cordon node X, reschedule the gang." Designing for that reader as well as for the steady-state one is what separates a scheduler that works from one that is operable.

**The ML engineer's view on P4.** The person actually running inference on your platform thinks in prefill, decode, KV cache and tokens per second — not in GPU allocations. A P4 answer that stays at the platform layer and never descends to why KV-cache memory rather than FLOPs is the binding constraint describes a platform an ML engineer would immediately distrust, because it optimises the wrong resource. Naming the constraint correctly is what proves you would be a good infrastructure partner rather than a generic ops layer bolted underneath.

**The synthesis view.** In a real loop these six are not silos. A P1 answer that never touches cost, or a P3 answer that never touches observability, misses the credit that only appears when you cross-reference your own reasoning — "as I said in the isolation discussion, this is the same dial one level up," or "and this is exactly the case where the utilisation metric can't help me, which is why I'd instrument step time instead." That cross-referencing is the cheapest available signal that you hold one model rather than six answers.

## Real-world use cases

- **Kubernetes v1.34 — Dynamic Resource Allocation graduated to GA.** The core DRA APIs in the
  `resource.k8s.io` group reached general availability in v1.34 and DRA is enabled by default from
  that release. **What it shows:** the direction accelerator allocation is moving — structured,
  claim-based allocation rather than an opaque integer resource count. Relevant to P3 because it
  changes what the scheduler knows, and to P2 because a claim's allocation result is a cleaner
  attribution source than a device-plugin ID string.

- **Meta, *Revisiting Reliability in Large-Scale Machine Learning Research Clusters*
  (arXiv:2410.21680, HPCA 2025).** Eleven months across two clusters (≈16K and ≈8K A100 GPUs).
  Measured MTTF of 7.9 hours for 1,024-GPU jobs; proactive lemon-node detection reduced the failure
  rate of jobs at 512 GPUs and above from about 14% to about 4%, identifying roughly forty faulty
  nodes at over 85% detection accuracy. **What it shows:** P5's central claim as a measured,
  published result rather than an intuition — and the figures to quote, with the discipline of
  saying which are measured and which (the larger-scale MTTFs) are the paper's projections.

- **NVIDIA `dcgm-exporter`, `etc/default-counters.csv` and the shipped Grafana dashboard.** The
  presence metrics ship enabled; the SM-breadth metrics ship commented out, and the vendor's own
  eight-panel dashboard has no SM-breadth panel. **What it shows:** P6's configuration argument, in
  one file. It also gives you a concrete, checkable sentence in a round rather than an assertion
  about what "most dashboards" do.

- **Prometheus `honor_labels` in `scrape_config`.** With the default (`false`), scraped labels that
  collide with target labels are renamed with an `exported_` prefix. **What it shows:** the
  mechanism behind P2's and P6's misattribution failure — a GPU cost dashboard that assigns the
  entire cluster's GPU-hours to the exporter's own namespace while looking completely reasonable.

- **vLLM's PagedAttention (SOSP 2023).** Fixed-size KV blocks with near-zero fragmentation, and
  iteration-level batching on top. **What it shows:** the mechanism behind P4's capacity argument —
  why the binding constraint is cache memory and why block allocation rather than contiguous arenas
  is what makes high concurrency possible.

- **OpenCost issues #3900 and #3828.** The maintainers' own record of the request-versus-usage cost
  basis gap, and an unaffiliated user asking for fractional-GPU costing. **What it shows:** P2's
  "why not just use the incumbent" follow-up, answerable with a citation rather than an opinion.

## Worked example

**A full P2 rep, timed, with the clock in the margin.** This is the shape to reproduce; the content
is §4 compressed to the block structure from §1.

```
   P2 · 35-MINUTE REP — what happens when
  ══════════════════════════════════════════════════════════════════════════════

   00:00  "Two clarifying questions: is this internal teams or external
          customers, and does anyone actually get billed, or is this
          visibility?"                             ← 2 questions, then stop
   00:40  THE UNIVERSAL OPENING. 1,000 GPUs · $2–3/GPU-hr · ~$20–25M/yr ·
          SLO = explainability, not accuracy · dominant failure mode =
          a disputed bill killing the programme · the dial = accuracy vs
          explainability.
   02:30  ── all four global dimensions banked, unprompted ──

   02:30  BLOCK B · the spine, shallow and complete:
            device counters → exporter → join at the UUID → two ledgers →
            rate card → per-tenant rows → showback surface
   08:00  "That's the whole system. Now the three places it's hard."

   08:00  BLOCK C · THE GPU LAYER — where the round is won
          C1  the two ledgers, and why they are never blended
          C2  the formula everyone writes, and its three flaws,
              volunteered:  util ≠ value · reserved-but-idle ·
              it rewards busy-looping
          C3  which metric for the efficiency lens, and why not the
              default one — with the 99 / 0.16 / 0.011 numbers
          C4  the sharing regimes: exact under MIG, estimate under
              time-slicing, and the exposure fraction QUANTIFIED
   20:00  ── at least three axes now at "2" ──

   20:00  BLOCK D · PRESSURE. Expect, in rough order of likelihood:
            "doesn't OpenCost do this?"        → source-level answer
            "how do you handle a dispute?"     → snapshot replay
            "how do you split a shared endpoint?" → compute-time split
            "what if a team games it?"         → billing on allocation
                                                  removes the incentive
   35:00  BLOCK E · close: build the join and the conservation check
          first; measure the exposure fraction; cut chargeback from v1.
          One honest limit: regime-3 attribution is an estimate and
          always will be.
```

**Self-scoring that rep:**

| Axis | Score | Why |
|---|---|---|
| telemetry source | 2 | named the pipeline *and* the two default traps (field off, label collision) |
| formula + flaws | 2 | volunteered three flaws before being asked, and rejected the formula as a billing basis |
| allocated vs utilised | 2 | two ledgers, never blended, with the reason |
| shared-endpoint split | 2 | compute-time split with the fallback named and recorded |
| showback → chargeback | 1 | mentioned the sequencing; did not defend the quarter-long delay |
| schema normalisation | 1 | named the standard and the two extensions; did not defend the column choice |
| dispute handling | 2 | snapshot replay, three dispute classes, conservation identity as the localiser |
| **global: scale/cost/SLO/failure** | **all volunteered by 02:30** | |

**Result: pass** — every axis ≥1, five axes at 2, all four global dimensions volunteered inside
three minutes. The remediation list is two items: rehearse the showback-duration defence, and be
able to justify the specific FOCUS columns rather than just naming the standard.

## Practice

1. **Run one thirty-five-minute recorded rep on each of P1–P6**, self-scored against that prompt's
   axes with the 0/1/2 rubric. Two full passes over the week.

2. **Re-drill your two lowest-scoring prompts** to two clean consecutive reps.

3. **Write your P2 card** — a one-page skeleton you can deliver cold in eight minutes. This is the
   prompt you volunteer, so it must be the one that needs no warm-up.

4. **Write your P4 card too**, because it is the trap prompt. Force yourself through prefill/decode,
   the KV-cache capacity calculation, and continuous batching until you can do the concurrency
   arithmetic out loud without hesitating.

5. **Drill the follow-ups separately from the answers.** Have someone read you the weak-versus-strong
   follow-ups from §3–§8 in random order and answer cold. The follow-ups are where the round is
   actually decided, and rehearsing an answer without rehearsing its defences is half the work.

6. **Add a reversal condition to every verdict** in every card. Then have someone push on the
   verdict and check that the reversal comes out without being asked for.

7. **Practise the cross-references.** In each rep, deliberately reach into another prompt at least
   once — P1 into P2 on cost, P3 into P6 on how you would detect the hang, P5 into P3 on drain and
   gang re-placement.

8. **Feed all six polished skeletons** into the [GPU platform capstone](../practice/gpu-platform-capstone/README.md)
   as the design-round appendix.

**Acceptance:** twelve recorded reps with scores logged · two clean reps on your weakest two prompts
· a P2 card deliverable cold in eight minutes · a P4 card containing the KV-cache concurrency
calculation · every verdict carrying a reversal condition · at least one cross-reference per rep.

## Common pitfalls

1. **Reciting the sharing mechanisms instead of choosing.** **Mechanism:** naming MIG, time-slicing
   and passthrough with their properties is a 1 on the axis; landing on which you would pick *here*
   and why is a 2. **Symptom:** the interviewer has to ask "so which would you use?"

2. **Describing gang scheduling without its cost consequence.** **Mechanism:** the mechanism is
   common knowledge; the sentence that scores is that you are billed for idle GPUs while the job
   hangs *and the utilisation metric reads full*, because the collective wait is a resident kernel.
   **Symptom:** a correct, forgettable explanation.

3. **Defining KV cache correctly and then answering capacity in FLOPs.** **Mechanism:** the two are
   different resources and only one runs out first in serving. **Symptom:** you describe paged
   attention accurately and then size the fleet by throughput.

4. **Answering P6 with more labels.** **Mechanism:** tenant × GPU × model is a cross product and
   Prometheus cardinality is a real budget. **Symptom:** you propose the labelling and the
   cardinality mitigation in separate breaths, or not at all — the senior version names both in one
   sentence.

5. **Waiting to be asked about cost.** **Mechanism:** cost volunteered is a 2 on a global dimension;
   cost prompted is a 1. **Symptom:** the phrase "have you thought about cost?" appears in your
   transcript.

6. **Going depth-first.** **Mechanism:** twenty minutes on the exporter means an incomplete system
   at the twenty-two-minute mark, so the interviewer never gets to push and Block D never happens —
   which is where most of the credit lives. **Symptom:** you never got to the interesting part and
   it felt like you ran out of time.

7. **A verdict with no reversal condition.** **Mechanism:** a verdict you cannot reverse is a
   preference, and preferences do not demonstrate that you weighed anything. **Symptom:** "why not
   X?" produces a defence of your choice rather than the boundary between the two.

8. **Treating the six prompts as separate.** **Mechanism:** the synthesis credit only exists in the
   cross-references. **Symptom:** six competent answers and a debrief that says "solid" rather than
   "clearly has one model of this."

## Self-check

- **What are the four global dimensions and when must they appear?** *Answer:* scale, cost, failure
  modes and SLOs, all four volunteered inside the first three minutes, before any interviewer
  prompt. They score differently depending on whether you were asked: an axis you were prompted into
  is evidence it was not part of your model. The universal opening exists so that all four are
  banked in one ninety-second block, after which your whole attention can go on the GPU layer.

- **In P1, what is the master dial, where does each sharing mode sit, and what does that imply for
  attribution?** *Answer:* isolation versus utilisation. Passthrough is maximum isolation and
  trivially exact attribution with the worst utilisation if the tenant cannot fill a card. MIG is
  hard isolation on SMs and framebuffer with exact attribution — an instance belongs to one
  container — at the cost of rigid partitions that do not rebalance and real stranding (7 × `1g.10gb`
  sums to about 0.858 of an A100 on a memory basis). MPS is a partial isolation via a thread-percentage
  cap, which bounds the attribution error. Time-slicing is maximum density, *no* isolation, a shared
  address space that is explicitly not a security boundary, and attribution that is an estimate only,
  because the counter is device-scoped and N tenants read one value. The same dial exists one level
  up as dedicated node pools versus a pooled cluster.

- **In P3, why does vanilla Kubernetes silently waste money on a distributed job, and what makes it
  invisible?** *Answer:* the scheduler binds pods independently with no concept of a gang, so a
  4-rank job can get three ranks running and one Pending. The three that started enter the first
  collective and block. No error is raised — the job reports Running — and because the collective
  wait is implemented as a resident kernel, all three idle GPUs report near-100% utilisation. So you
  are billed for three idle GPUs indefinitely while every dashboard says they are busy. The only
  visible signature is that tensor activity collapsed while the presence metric stayed pinned. The
  fix is all-or-nothing admission, with preemption evicting the whole gang, because evicting one pod
  recreates the same state.

- **In P4, what is the binding capacity constraint and how would you compute it?** *Answer:*
  KV-cache memory, not FLOPs. Start from the card's HBM — 80 GB on an H100 — subtract the weights
  (an 8B model in BF16 is 16 GB), subtract framework and activation overhead, and the remainder is
  the KV-cache arena. Per-sequence cache is `2 × layers × kv_heads × head_dim × bytes × seq_len`,
  where the 2 is keys plus values, so concurrency is the arena divided by that, and it falls roughly
  linearly as context length grows. This is why a long-context tenant costs many times a
  short-context one for the same request count, and why paged block allocation matters — it stops
  fragmentation rather than raw capacity from limiting concurrency.

- **In P5, what is a lemon node, why is it hard to detect, and what is it worth?** *Answer:* a node
  that passes point-in-time health checks and repeatedly kills large jobs. It is hard to detect
  precisely because it passes those checks, so the signal has to be *associative* — the failure rate
  of jobs that touched a node, compared against the fleet baseline over a window — which is a
  statistical claim needing enough job-touches per node and careful handling of the confound that
  large jobs touch more nodes and fail more often anyway. The published value: proactive lemon-node
  detection reduced the failure rate of jobs at 512 GPUs and above from about 14% to about 4% on two
  large research clusters, identifying roughly forty faulty nodes at over 85% detection accuracy
  (arXiv:2410.21680). The related trap to name unprompted is "passes idle, fails under load" — the
  load-level diagnostic is a stress test that cannot be run on a node serving a job.

- **In P6, name the alerting policy in four rules.** *Answer:* (1) Page on sustained near-zero
  SM-breadth, **gated on framebuffer**, because a paused-but-loaded serving replica has near-zero
  activity and tens of gigabytes resident, and reclaiming it destroys a warm replica and gets the
  reclaim system switched off politically. (2) Warn, never page, on tensor activity — efficiency is a
  conversation, not an incident. (3) Never alert on the presence metrics at all, because they read
  near 100 on exactly the expensive waste cases — batch-1 decode, spin-waits, blocked collectives,
  starved dataloaders — so such a rule never fires on your most expensive waste. (4) Alert on
  **absence**: profiling fields are dropped rather than zeroed when unavailable, and absence is not
  zero in PromQL, so an average silently skips the missing GPU and a fleet where half the cards went
  dark shows an unchanged healthy average.

- **What does a "2" sound like versus a "1" on any axis?** *Answer:* a 1 names the thing; a 2 names
  the thing, states which option you chose *here*, gives the reason in terms of a constraint you
  established earlier in the round, and names the condition under which you would reverse it. "We'd
  use MIG for isolation" is a 1. "MIG here, because attribution has to survive a chargeback dispute
  and only a hardware partition gives an exact split — I'd trade the ~14% stranding for that, and
  I'd reverse it if the tenant mix got bursty enough that static geometry stranded more than
  time-slicing wastes" is a 2. The reversal condition is the part most candidates omit and it is the
  part that proves the verdict was reasoned rather than defaulted.

## Connections & what's next

P1–P6 pull directly on the numbers and artifacts from modules 01–11, and on lesson 02's
rejected-alternatives bank — the verdicts and reversal conditions you wrote there are the same ones
you deliver under a clock here. Lesson 03's design doc is the written form of a P2 answer, which is
why the Alternatives and Risks sections drill so directly into this round. Lesson 04's objection map
is the same material as the follow-up tables above, aimed at a reader rather than an interviewer.
Forward: lesson 07 narrates the concrete artifacts that back these designs, and lesson 09 runs P1–P6
under full-loop fatigue rather than in isolation.

Next: [06 — Debugging drills](06-debugging-drills.md), which drills the incident and
live-terminal round — a harder-to-fake companion to the whiteboard design round you just drilled,
and the one round where the util-versus-work reflex is tested directly rather than described.

## References & further reading

**Primary sources**

- Kubernetes blog — *Kubernetes v1.34: DRA has graduated to GA* — https://kubernetes.io/blog/2025/09/01/kubernetes-v1-34-dra-updates/ — read for: which `resource.k8s.io` APIs reached GA, and that DRA is enabled by default from v1.34. The citable fact for P3's "where this is heading" beat.
- Kueue documentation — cohorts and quota borrowing — https://kueue.sigs.k8s.io/docs/concepts/cohort/ — read for: the precise mechanism name and semantics behind P3's borrowing-and-reclaim answer.
- vLLM / PagedAttention (SOSP 2023) — https://dl.acm.org/doi/10.1145/3600006.3613165 — read for: block-based KV-cache allocation and iteration-level batching, the mechanisms behind P4's capacity and throughput answers.
- Meta — *Revisiting Reliability in Large-Scale Machine Learning Research Clusters* (arXiv:2410.21680, HPCA 2025) — https://arxiv.org/abs/2410.21680 — read for: the measured 7.9-hour MTTF at 1,024 GPUs and the lemon-node result (large-job failure rate from ~14% to ~4%, ~40 nodes at >85% accuracy) that P5 rests on. Note which figures are measured and which are projections.
- NVIDIA `dcgm-exporter`, `etc/default-counters.csv` — https://github.com/NVIDIA/dcgm-exporter/blob/main/etc/default-counters.csv — read for: P6's configuration argument — the honest field ships commented out.
- Prometheus `scrape_config` / `honor_labels` — https://prometheus.io/docs/prometheus/latest/configuration/configuration/ — read for: the `exported_` renaming behind the misattribution failure in P2 and P6.
- FinOps Foundation — FOCUS specification — https://focus.finops.org/ — read for: the split-cost-allocation columns P2 uses and the two gaps that justify its extensions.
- OpenCost issues #3900 and #3828 — https://github.com/opencost/opencost/issues/3900 · https://github.com/opencost/opencost/issues/3828 — read for: the citable answer to P2's "doesn't a tool already do this" follow-up.

**Course-internal sources**

- `platform-eng/modules/05-gpu-observability/lessons/01-lie-and-truth.md` — the field semantics, the batch-1 derivation, and the alerting rules quoted throughout P4 and P6.
- `platform-eng/modules/04-gpu-on-kubernetes/lessons/10-capstone-per-pod-attribution.md` — the share model, the MIG stranding arithmetic, and the exposure fraction used in P1 and P2.
- `platform-eng/modules/11-gpu-cost-economics/lessons/09-existing-tooling-limits.md` — the source-level answer to P2's incumbent-tool follow-up.

**Not relied upon**

- Vendor blog posts on multi-tenant cluster design, GPU sharing and cost attribution reference
  architectures were consulted while assembling the probe-axis lists. They are useful as evidence
  that these six prompts reflect real product surfaces, but no number or mechanism above is taken
  from them, and no claim is made about any named company's interview process — the six prompts are
  presented as the recurring problem set of this market, not as any employer's question bank.
