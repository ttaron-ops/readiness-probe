---
lesson: "02b.7"
title: "Power and thermals as measurable undelivered FLOPs"
module: "02b"
concept: "Power and thermals as measurable undelivered FLOPs"
status: not-started
est_time: "4.5h"
prev: "06-storage-nvme.md"
next: "08-capstone-topology-teardown.md"
artifacts: []
sources: 7
---
# 02b.7 · Power and thermals as measurable undelivered FLOPs
> **Concept.** A throttled clock is paid-for-but-undelivered compute — power caps and thermal limits force DVFS clock-downs that `nvidia-smi` will name for you, while utilization keeps reading 100%. At fleet scale the same phenomenon shows up a level up, as synchronized power swings that stress the building, not just the die.
>
> Module: [🧬 02b — Host hardware and topology](../README.md) · Deliverable: [Topology Teardown](../practice/topology-teardown/README.md)

## Where this fits

Lesson 6 established that a data path can be topologically correct on paper (right NUMA node, right PCIe switch, GDS enabled) and still under-deliver if the pipe itself is starved or misconfigured — you learned to distrust "GDS enabled" until `gdscheck -p` confirms the fast path is actually active. This lesson applies that same distrust-the-green-light discipline to a different resource: clock frequency. Everything through lesson 6 assumed the GPU, once data reaches it, runs at its rated clock. It usually doesn't. Power and thermal governors silently pull the clock down below boost, and unlike a misrouted PCIe lane, this cause is *named for you* in a bitmask almost nobody reads. This lesson closes the last gap before the capstone: after this, you have every host-side cause of "GPU busy, throughput low" in hand — NUMA, PCIe, storage placement, and now power/thermal — which is exactly the diagnostic tree lesson 8 asks you to walk from symptoms alone.

## Why this matters

Utilization is a liar about clocks. `nvidia-smi` can report `GPU-Util: 100%` while the SM clock is pinned 300–500 MHz below its boost ceiling because the GPU hit a power cap or a thermal limit. Those megahertz are FLOPs you are paying for and not receiving. Unlike a data stall, this waste is *directly observable and named*: the GPU tells you exactly why it slowed down via `clocks_throttle_reasons`. Almost nobody reads it.

The differentiator here is that you can put a number on it. An H100 SXM boosts to ~1980 MHz. If it's running at 1590 MHz under a power cap, that's ~20% fewer cycles per second on the tensor cores — a ~20% throughput haircut on a $2–3/GPU-hr asset that the utilization dashboard renders as "fully busy." Across a rack that difference is the gap between the per-GPU spec you budgeted for and the per-GPU delivery you actually get. Knowing which throttle reason is *intentional* (you set a power cap for grid/rack reasons) versus *a cooling failure* (you're paying for heat removal you don't have) is the difference between a defensible capacity plan and silent burn.

The stakes are not only per-node. As you'll see below, production ML fleets discovered that synchronized training-step power ramps across thousands of accelerators can swing facility-level power draw by tens of megawatts, near-instantaneously — a problem that damages shared electrical infrastructure (rectifiers, transformers, UPS batteries) independent of any single chip's thermal state. That's a capital-equipment-lifecycle cost, not just a throughput one, and it's exactly the kind of fact a staff-level interviewer at a neocloud or hyperscaler expects you to already know.

## What's new here (calibration)

- **You already know:** CPUs do DVFS (frequency/voltage scaling), thermal throttling exists as a concept, `nvidia-smi` shows temperature and power draw at a glance. You've dealt with CPU `turbo`/`C-states` and RAPL on the Linux side, so the *mechanism* of frequency scaling under a physical constraint is not new.
- **What's genuinely new:** Reading the GPU's *reason* bitmask — `clocks_throttle_reasons` distinguishes SW power cap, HW power brake, SW thermal slowdown, and HW thermal slowdown, and each has a different owner and cost story. The fact that H100 SXM is a ~700 W part and that rack power density (8 GPUs ≈ 5.6 kW of GPU alone, ~10+ kW/node all-in) forces liquid cooling. The systems reality that a *fully populated* rack underperforms per-GPU spec because shared power and shared cooling become the binding constraint before the silicon does. And, new for this rewrite: the fleet-scale phenomenon of *synchronized* power ramping across thousands of accelerators, the adjacent Xid/SXid hardware-fault diagnostic layer that a throttle reading alone can't distinguish from, and the production health-check practice (`dcgmi` + `dmesg` sweeps) that operationalizes all of this continuously rather than as one-off debugging.

## Core concepts

### DVFS, power caps, and thermal limits

A GPU has a boost clock it *wants* to run at and several governors that pull it down:

- **Power cap (enforced limit).** The GPU keeps clock × voltage such that board power stays under the enforced limit (`Enforced Power Limit`, settable with `nvidia-smi -pl`). Hit the cap and DVFS drops the SM clock to fit the power budget. This is *intentional* when you set it (rack/PDU/grid constraint) and shows as `SW Power Cap: Active`. There's also a hardware fast-trip, the **power brake** (`HW Power Brake Slowdown`), asserted by external circuitry (e.g. a PDU/EDPp signal) — that one is the facility telling the GPU to back off *now*.
- **Thermal slowdown.** Two tiers. `SW Thermal Slowdown` engages as the GPU approaches its thermal limit (GPU temp or, on HBM stacks, memory temp) and DVFS steps the clock down to hold temperature. `HW Thermal Slowdown` is the emergency hardware trip near the shutdown threshold — a much harder, deeper clock cut to avoid damage. SW thermal = "cooling is marginal"; HW thermal = "cooling is failing."

The key framing: **a power cap is a decision, a thermal slowdown is a symptom.** If you're power-capped, you chose that trade (or your facility did) and you can reason about it. If you're thermally slowed, you are paying for cooling capacity you don't actually have — the heat isn't leaving fast enough, so the silicon protects itself by delivering fewer FLOPs. None of this is a "bug" — it's DVFS, the chip protecting itself from a physical constraint, working exactly as designed. The question a platform engineer asks is never "why is this broken," it's "which constraint is binding, and did I choose it."

### Reading the throttle reasons

`nvidia-smi -q -d PERFORMANCE,POWER` dumps both the throttle bitmask and the power/clock state. The block you care about:

```
    Clocks Throttle Reasons
        Idle                          : Not Active
        Applications Clocks Setting   : Not Active
        SW Power Cap                  : Active
        HW Slowdown                   : Not Active
            HW Thermal Slowdown       : Not Active
            HW Power Brake Slowdown   : Not Active
        SW Thermal Slowdown           : Not Active
        Sync Boost                    : Not Active
```

Reads: this GPU is clock-limited *by a software power cap* and nothing else — intentional if you set the cap. If `SW Thermal Slowdown : Active` appeared, that's marginal cooling. If `HW Thermal Slowdown : Active`, that's a cooling emergency and you should be paging someone. `Sync Boost : Active` means it's being held to match peers in a synchronized-boost group (common in DGX/HGX so all 8 GPUs step together).

You can also query just the mask, which is scriptable for DCGM/Prometheus exporters (the bitmask field names are documented in NVML's `nvmlClocksThrottleReasons` group — see References):

```
$ nvidia-smi --query-gpu=clocks_throttle_reasons.active,clocks.sm,clocks.max.sm,power.draw,enforced.power.limit,temperature.gpu --format=csv
0x0000000000000004, 1590 MHz, 1980 MHz, 699.8 W, 700.0 W, 71
```

`0x4` is the `SW Power Cap` bit. Power draw is pinned at the 700 W enforced limit, and the SM clock sits at 1590 against a 1980 max — that's the ~20% you're leaving on the table, and the reason bit tells you it's the cap, not heat.

### The physical envelope

- **H100 SXM5 ≈ 700 W** per GPU (the PCIe variant is ~350 W; SXM is the datacenter part). Eight of them is ~5.6 kW of GPU before you count CPUs, NICs, DPUs, NVSwitch, and fans — an HGX/DGX H100 node draws on the order of ~10 kW.
- At ~10 kW/node, a rack with several nodes is **40–80+ kW**. That density is why air cooling runs out of headroom and the industry moved to **direct-to-chip liquid cooling** (and, at the extreme, immersion). Air simply can't pull ~700 W off each of eight closely-packed dies in a hot aisle without the inlet temperature climbing until SW thermal slowdown kicks in.

### Why a fully populated rack underperforms per-GPU spec

Name the two mechanisms:

1. **Shared power budget.** The rack PDU / branch circuit has a finite amperage. Populate every slot and aggregate draw approaches the breaker limit, so operators set a per-GPU or per-node **power cap** to stay under it — every GPU runs capped (`SW Power Cap` / possibly `HW Power Brake`) below its boost clock. The rack is oversubscribed on watts, so no single GPU gets its full envelope.
2. **Shared cooling / thermal coupling.** Eight hot GPUs raise the local inlet/coolant temperature; a full rack raises it more. As inlet climbs toward the thermal limit, `SW Thermal Slowdown` clock-downs appear — first on the hottest GPUs (often the middle of the chassis or the top of the rack). Each GPU's heat is a neighbor's cooling problem, so density itself lowers the achievable clock.

Both are invisible per-GPU utilization-wise: every GPU reads "busy," but the rack delivers less aggregate throughput than `8 × single-GPU-spec` because watts and cooling — not silicon — are the binding constraint. That gap is the cost story.

### Beyond one GPU: synchronized fleet-wide power ramping

Everything above is a *steady-state* story: one GPU, sustained load, a clock that settles at a capped or thermally-limited value. Production ML fleets have surfaced a second, distinct phenomenon that only shows up at scale: **synchronized power ramping**. Training workloads have a step structure — compute-heavy phases (forward/backward pass) alternate with comparatively idle phases (gradient all-reduce, optimizer step, data loading gaps). Because a whole fleet of accelerators executes the same step boundaries in near-lockstep (that's what data-parallel/tensor-parallel training *is*), thousands of chips ramp from idle to peak power and back again almost simultaneously.

Google Cloud's engineering team documented this directly: synchronized ML power draw can swing a cluster's electrical demand by **tens of megawatts, near-instantaneously** — a load profile unlike traditional, uncorrelated datacenter workloads, and one that stresses rectifiers, transformers, and UPS batteries that weren't designed for that rate of change. Their fix was a compiler-level technique: detect the workload's power "signature" and reshape *when* different compute blocks run within a step to smooth the aggregate power curve, rather than letting every chip ramp in lockstep. Measured results: **roughly a 50% reduction in cluster-level power fluctuation**, and **per-chip hotspot temperature swings reduced from roughly 20°C to roughly 10°C**, at **under 1% performance overhead**. (This work is TPU-specific, but the mechanism — smoothing compute-block activity around synchronization points to flatten the power draw curve — is a general, transferable idea, not an artifact of one vendor's silicon.)

The reframe this earns you: `nvidia-smi -q -d PERFORMANCE,POWER` on one GPU can never show you this. It's an aggregate, cluster-level fact about *when* thousands of chips draw power together, not about any single chip's instantaneous state. A staff engineer who only checks per-GPU throttle reasons has covered the die-level story and missed the facility-level one.

### The adjacent diagnostic layer: Xid/SXid errors

Throttling (this lesson's focus) and **Xid errors** are complementary but distinct signals, and conflating them sends you down the wrong remediation path. A throttle reason is DVFS working correctly — a policy response to a constraint. An **Xid error**, surfaced in `dmesg` by the NVIDIA driver, is a hardware/driver *fault* event: something actually broke or misbehaved. A classic critical example is **Xid 79, "GPU fell off the bus"** — often triggered by thermal stress or power fluctuation in dense racks, but it is not the same thing as a throttle bit being set. **SXid** is the NVSwitch-specific equivalent, relevant in multi-GPU/NVSwitch systems where the switch fabric itself, not a GPU, is the faulting component.

Why this matters diagnostically: if you see elevated temperatures and a `SW Thermal Slowdown` bit, the fix is a cooling/power conversation. If you see an Xid 79 in `dmesg`, the fix is a hardware repair ticket — the node needs to be cordoned, not re-tuned. The two can share a root cause (thermal stress in a dense rack can produce both a throttle event and, if things go far enough, an Xid), but they are different rungs on the same ladder, and only one of them means "file a hardware ticket."

### Operationalizing this at fleet scale

Production shops don't wait for a job to report low throughput before checking power/thermal state — they run **passive health checks** continuously. A practical pattern (documented in Modal's fleet-operations writeup, cited below) is a periodic sweep combining `dcgmi` diagnostics with `dmesg` scanning, specifically looking for thermal violations, sync-boost violations, hardware slowdowns, and excessive temperatures — one production writeup cites **>88°C** as a practical operational red-flag threshold worth alerting on before scheduling new work onto a node. This is the fleet-scale version of the single-GPU `nvidia-smi -q` read this lesson teaches: same signal, checked continuously and automatically instead of manually per-incident. It's also a direct preview of what lesson 8's capstone Part B asks you to design: an exporter/alert built from exactly this kind of signal.

## Perspectives

**Developer.** Throttling is invisible from training code — a developer sees only "throughput is lower than expected," with no signal distinguishing power/thermal from a data or communication bottleneck. Framework-level profilers report step time, not clock state; telling the two apart requires the platform-level tools this lesson teaches, not application profiling.

**Operator / facilities.** Power and cooling are *shared, finite* resources at the rack/row/facility level. An operator's job includes setting deliberate power caps to protect the PDU/breaker, and then reading whether *observed* throttling matches the *intended* cap — a mismatch (e.g. seeing thermal throttling even though the power budget has headroom) means the cooling design itself is under-provisioned for the deployed density, a capacity-planning finding, not a per-job bug.

**Hardware / silicon.** DVFS is the chip protecting itself. Every throttle reason is the silicon trading clock frequency for staying within a physical constraint — power delivery capacity or junction/HBM temperature. None of it is a fault; it's the intended safety mechanism working correctly. That reframes "why is my GPU throttled" from a fault-finding question into a capacity-planning question, and reframes an Xid error as the genuinely different case where something *did* break.

**Economics / facility design.** Synchronized ML power draw is different enough from traditional datacenter load profiles that it damages shared electrical equipment and burns UPS battery life over time — power/thermal management is now a **capital equipment lifecycle** cost question, not just a per-job throughput question. A platform engineer who can speak to both the per-GPU dollar cost of a throttled clock *and* the facility-level cost of an unshaped power curve is arguing from the full picture an infra-economics interviewer wants to hear.

## Real-world use cases

- **Google Cloud — "Mitigating power and thermal fluctuations in ML infrastructure"**: documents cluster-level synchronized power swings of tens of megawatts and a compiler-level workload-signature technique that cut power fluctuation by ~50% and per-chip hotspot temperature swings from ~20°C to ~10°C, at under 1% performance overhead. Shows the facility-level version of this lesson's die-level story, with a concrete, measured software mitigation. https://cloud.google.com/blog/topics/systems/mitigating-power-and-thermal-fluctuations-in-ml-infrastructure
- **Modal — "Keeping 20,000 GPUs healthy"**: documents the Xid/SXid error taxonomy in production, names Xid 79 ("GPU fell off the bus"), and describes the `dcgmi` + `dmesg` passive health-check pattern — including an ~88°C operational temperature threshold — used to catch thermal and hardware issues before they hit a running job. Shows the adjacent hardware-fault diagnostic layer and how it's operationalized continuously at fleet scale. https://modal.com/blog/gpu-health
- **Meta Engineering — "How Meta keeps its AI hardware reliable"**: describes Meta's FBAR (Facebook Auto Remediation) automated-remediation system and names thermal runaway explicitly as a transient-error class it detects and mitigates at fleet scale, alongside GPU/HBM3 failures. Shows the reliability-engineering framing this lesson's throttle/Xid distinction feeds into. https://engineering.fb.com/2025/07/22/data-infrastructure/how-meta-keeps-its-ai-hardware-reliable/

## Worked example

Put a load on the GPU, read the throttle state, then *cause* a throttle and watch the clock move — then reason about what happens one level up, at rack scale.

Baseline under load (a training step or `dcgmproftester`/`gpu-burn` for a synthetic load):

```
$ nvidia-smi --query-gpu=clocks.sm,power.draw,enforced.power.limit,temperature.gpu,clocks_throttle_reasons.active --format=csv,noheader
1980 MHz, 642.1 W, 700.00 W, 68, 0x0000000000000000
```

Read this as: clock at boost (1980), power headroom (642 < 700), cool (68°C), no throttle bit active — the GPU is delivering full spec. Good baseline.

Now impose a power cap and watch DVFS respond:

```
$ sudo nvidia-smi -pl 400
Power limit for GPU 00000000:C1:00.0 was set to 400.00 W from 700.00 W.

$ nvidia-smi --query-gpu=clocks.sm,power.draw,enforced.power.limit,clocks_throttle_reasons.active --format=csv,noheader
1350 MHz, 399.6 W, 400.00 W, 0x0000000000000004
```

Trace it: the enforced limit is now 400 W, power draw pins right at it (399.6), the SM clock collapsed from 1980 → 1350 MHz (~32% fewer cycles), and the reason bit is `0x4` = `SW Power Cap: Active`. You just made the throttle happen on purpose and watched the clock/throughput fall in lockstep with the cap. If you were running a benchmark, its samples/sec would drop by a similar proportion — that's the direct clock→throughput correlation.

Restore the cap when done:

```
$ sudo nvidia-smi -pl 700
```

Conclusion for the deliverable: *at 700 W cap the GPU held 1980 MHz (full spec). Capping to 400 W drove `SW Power Cap: Active` (0x4) and dropped SM clock to 1350 MHz (~32%), with a proportional throughput loss — demonstrating that a power cap is a directly measurable, intentional trade of FLOPs for watts.* If instead you'd captured `SW Thermal Slowdown` at full power limit, the same clock drop would be *unpaid-for cooling debt*.

**The rack-scale extension.** Now reason past the single GPU: if this box's seven neighbors in the rack are all training the same job and hit the same step boundary in lockstep, they all ramp from idle to ~700 W in roughly the same instant. Even if *every individual GPU's* `enforced.power.limit` is respected — none of them ever shows `SW Power Cap: Active` on a per-sample basis — the rack's PDU sees a near-simultaneous multi-kilowatt swing, repeated every step, for the whole run. That's the Google Cloud phenomenon in miniature: no single-GPU telemetry catches it, because it's a property of *when* the ramps happen in aggregate, not of any one chip's steady-state clock. This is why fleet-scale monitoring, not just per-node `nvidia-smi -q`, is the production answer — the single-GPU worked example above is necessary but not sufficient evidence at scale.

## Practice

Feeds the **Topology Teardown** deliverable.

1. Put a sustained load on one GPU (real training step, or `gpu-burn` / `dcgmproftester`). Run `nvidia-smi -q -d PERFORMANCE,POWER` and locate the `Clocks Throttle Reasons` block plus current SM clock, max SM clock, power draw, enforced power limit, and temperature.
2. Record the baseline: is any throttle reason `Active`? What's `clocks.sm` vs `clocks.max.sm`?
3. Lower the power cap: `sudo nvidia-smi -pl <below the load's draw>` (e.g. 400 on a 700 W part). Re-read the throttle block and the SM clock. Confirm `SW Power Cap: Active` and record the new clock. If you have a throughput number (samples/sec, tokens/sec), record before/after.
4. Restore: `sudo nvidia-smi -pl 700` (or the card's default). Note whether you ever observed a *thermal* reason at full power — if so, that's cooling debt, not an intentional cap.
5. Check the adjacent diagnostic layer: run `dmesg | grep -i xid` (and, on a multi-GPU NVSwitch box, look for SXid entries too). Confirm you see none during the exercise — and note in your write-up that a real Xid 79 during a throttle investigation would change the fix from "retune the cap" to "file a hardware ticket."

**Acceptance:** a note in the Topology Teardown showing at least one throttle reason observed (with its bit/name), the clock delta it caused (SM MHz before/after), and, where available, the throughput impact — plus one sentence classifying it as intentional (power cap) vs symptomatic (thermal), and one sentence confirming `dmesg` showed no Xid/SXid faults during the run (or documenting one if it did). Paste the relevant `nvidia-smi -q` lines.

## Common pitfalls

1. **Treating any throttle reason as a bug.** Every bit is DVFS working correctly under a real constraint. The question is never "is this broken," it's "which constraint is binding, and is it one I chose?"
2. **Trusting `GPU-Util: 100%` as proof of full throughput.** Utilization only means a kernel was resident in the sample window — it says nothing about clock. Always pair it with `clocks.sm` vs `clocks.max.sm`.
3. **Assuming a fully populated rack delivers `N × single-GPU spec`.** Shared power and shared cooling are frequently the binding constraint before silicon is, and neither shows up in per-GPU utilization.
4. **Conflating power/thermal throttling with an Xid/SXid hardware fault.** Throttling is a policy response (DVFS); an Xid/SXid error (e.g. code 79, "fell off the bus") is a fault event. Related — both can stem from thermal/power stress in dense racks — but diagnostically distinct, and conflating them sends you toward the wrong fix: retuning a power cap when you actually need a hardware repair ticket, or vice versa.
5. **Assuming per-GPU throttle telemetry is sufficient at fleet scale.** The interesting failure mode at scale is *synchronized*, cluster-level power/thermal swings that no single GPU's `clocks_throttle_reasons` fully captures. Fleet-level or at minimum rack-level power telemetry is needed to see the aggregate pattern a single-node worked example can't show.

## Self-check

**(a)** Which throttle reason means "you're paying for cooling you don't have" vs "you capped power on purpose"?

**Answer:** `SW Thermal Slowdown` (and its emergency tier `HW Thermal Slowdown`) means the GPU is clock-limited because it can't shed heat fast enough — you're paying for cooling capacity you don't actually have. `SW Power Cap` (and the facility-driven `HW Power Brake`) means the clock is limited to stay under an enforced power limit that you or the facility set on purpose. Thermal = symptom of a cooling shortfall; power cap = an intentional watts-for-FLOPs trade.

**(b)** Utilization reads 100% but clocks are capped — is the GPU "busy," and what's the cost story?

**Answer:** It's "busy" only in the sense that a kernel is resident every sample window — that's all `GPU-Util` measures. It is *not* delivering full throughput: at a capped/throttled clock it does fewer cycles per second, so tensor-core FLOPs/sec are down proportional to the clock deficit (e.g. 1590 vs 1980 MHz ≈ 20% fewer). The cost story: you pay full GPU rate for a GPU delivering, say, 80% of spec, and utilization hides it. The undelivered 20% is real money — only `clocks.sm` vs `clocks.max.sm` and the throttle reason expose it.

**(c)** Why does a fully populated rack underperform per-GPU spec — name two mechanisms?

**Answer:** (1) **Shared power budget** — the PDU/branch circuit is finite, so a fully populated rack is capped per-GPU/per-node to stay under the breaker (`SW Power Cap`/`HW Power Brake`), holding every GPU below its boost clock. (2) **Shared cooling / thermal coupling** — each GPU's heat raises the local inlet/coolant temperature, so at full density GPUs approach the thermal limit and hit `SW Thermal Slowdown`. Both drop clocks while per-GPU utilization still reads 100%, so aggregate throughput falls short of `8 × single-GPU spec` because watts and cooling, not silicon, are the binding constraint.

**(d)** What's the difference between a GPU throttling under `clocks_throttle_reasons` and a GPU throwing an Xid 79 error, and why does the distinction matter for remediation?

**Answer:** A throttle reason is DVFS working as designed — the silicon voluntarily trading clock for staying within a power or thermal constraint; the fix is a power-cap or cooling conversation. Xid 79 ("GPU fell off the bus") is a hardware/driver *fault* event surfaced by the NVIDIA driver in `dmesg` — something actually broke, even though thermal or power stress can be the trigger. The distinction matters because they route to different remediations: retune the cap or improve cooling for a throttle reason; cordon the node and file a hardware repair ticket for an Xid fault. Treating one as the other wastes time chasing the wrong fix.

**(e)** Why can synchronized ML training workloads stress datacenter electrical infrastructure differently than traditional, uncorrelated datacenter load, even when every individual accelerator stays under its power cap?

**Answer:** Data-/tensor-parallel training makes thousands of accelerators hit the same step boundaries in near-lockstep, so they ramp from idle to peak power together rather than at independent, staggered times as traditional uncorrelated workloads do. The result is a near-instantaneous, tens-of-megawatt aggregate power swing at the cluster level — a load-profile shape that rectifiers, transformers, and UPS batteries weren't designed for, even though no single chip ever exceeds its individual `enforced.power.limit`. It's an aggregate, timing-driven fact that per-GPU telemetry cannot see.

## Connections & what's next

Power and thermal throttling is the last of the four host-side "GPU busy, throughput low" causes this module builds — after NUMA placement (lessons 1–2), PCIe link health (lesson 3), topology-aware scheduling (lesson 5), and storage placement (lesson 6). It's also the one cause with a second, distinct diagnostic layer (Xid/SXid) that this lesson's pitfalls section explicitly separates from throttling proper — remember both when you get to lesson 8's diagnostic tree.

Next is the **capstone**: reconcile all four topology tools into one diagram on a real node, then measure what a misalignment costs. Lesson 8's self-check symptom-list ("GPU at 100%, throughput half spec, no error") is the full checklist this module has been building toward — NUMA, PCIe, GPU↔NIC path, NVMe placement, and now power/thermal (plus, as a distinct seventh hypothesis, a hardware fault surfaced via Xid/SXid) all belong on that list, and this lesson is where the last two entries came from.

## References & further reading

**Primary sources**
- NVIDIA DGX H100 — Power Capping — the authoritative doc on setting and reasoning about enforced power limits on H100-class systems. https://docs.nvidia.com/dgx/dgxh100-user-guide/power-capping.html
- `nvidia-smi` manual / `-q` device query reference — the full list of `-d` query groups (`PERFORMANCE`, `POWER`, `CLOCK`) and the `clocks_throttle_reasons.*` fields for scripting. https://docs.nvidia.com/deploy/nvidia-smi/index.html
- NVML API Reference, `nvmlClocksThrottleReasons` group — the exact bitmask/field names behind `clocks_throttle_reasons`, useful when building an exporter. https://docs.nvidia.com/deploy/nvml-api/group__nvmlClocksThrottleReasons.html

**Real-world engineering blogs**
- Google Cloud, "Mitigating power and thermal fluctuations in ML infrastructure" — synchronized fleet-wide power ramping, the ~50% fluctuation reduction and ~20°C→~10°C hotspot-swing reduction from compiler-level workload shaping. https://cloud.google.com/blog/topics/systems/mitigating-power-and-thermal-fluctuations-in-ml-infrastructure
- Modal, "Keeping 20,000 GPUs healthy" — Xid/SXid taxonomy, Xid 79, and the `dcgmi`+`dmesg` passive fleet health-check pattern. https://modal.com/blog/gpu-health
- Meta Engineering, "How Meta keeps its AI hardware reliable" — FBAR automated remediation and thermal runaway as a named fleet-scale failure class. https://engineering.fb.com/2025/07/22/data-infrastructure/how-meta-keeps-its-ai-hardware-reliable/

**Deeper dives**
- Gregg, *Systems Performance*, 2nd ed. — the CPU/thermal DVFS and power-management chapters for the general model of frequency scaling under power/thermal limits; read the GPU as a fast-DVFS instance of the same mechanism. https://www.brendangregg.com/systems-performance-2nd-edition-book.html
