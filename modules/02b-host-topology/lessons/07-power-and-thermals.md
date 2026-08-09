---
lesson: "02b.7"
title: "Power and thermals as measurable undelivered FLOPs"
module: "02b"
concept: "Power and thermals as measurable undelivered FLOPs"
status: not-started
est_time: "2.5h"
artifacts: []
---
# 02b.7 · Power and thermals as measurable undelivered FLOPs
> **Concept.** A throttled clock is paid-for-but-undelivered compute — power caps and thermal limits force DVFS clock-downs that `nvidia-smi` will name for you, while utilization keeps reading 100%.
>
> Module: [🧬 02b — Host hardware and topology](../README.md) · Deliverable: [Topology Teardown](../practice/topology-teardown/README.md)

## Why this matters

Utilization is a liar about clocks. `nvidia-smi` can report `GPU-Util: 100%` while the SM clock is pinned 300–500 MHz below its boost ceiling because the GPU hit a power cap or a thermal limit. Those megahertz are FLOPs you are paying for and not receiving. Unlike a data stall, this waste is *directly observable and named*: the GPU tells you exactly why it slowed down via `clocks_throttle_reasons`. Almost nobody reads them.

The differentiator here is that you can put a number on it. An H100 SXM boosts to ~1980 MHz. If it's running at 1590 MHz under a power cap, that's ~20% fewer cycles per second on the tensor cores — a ~20% throughput haircut on a $2–3/GPU-hr asset that the utilization dashboard renders as "fully busy." Across a rack that difference is the gap between the per-GPU spec you budgeted for and the per-GPU delivery you actually get. Knowing which throttle reason is *intentional* (you set a power cap for grid/rack reasons) versus *a cooling failure* (you're paying for heat removal you don't have) is the difference between a defensible capacity plan and silent burn.

## What's new for you

- **You know:** CPUs do DVFS (frequency/voltage scaling), thermal throttling exists, `nvidia-smi` shows temperature and power draw. You've dealt with CPU `turbo`/`C-states` and RAPL on the Linux side.
- **What's new:** Reading the GPU's *reason* bitmask — `clocks_throttle_reasons` distinguishes SW power cap, HW power brake, SW thermal slowdown, and HW thermal slowdown, and each has a different owner and cost story. The fact that H100 SXM is a ~700 W part and that rack power density (8 GPUs ≈ 5.6 kW of GPU alone, ~10+ kW/node all-in) forces liquid cooling. And the systems reality that a *fully populated* rack underperforms per-GPU spec because shared power and shared cooling become the binding constraint before the silicon does — which is exactly the effect no per-GPU utilization metric surfaces.

## Core notes

### DVFS, power caps, and thermal limits

A GPU has a boost clock it *wants* to run at and several governors that pull it down:

- **Power cap (enforced limit).** The GPU keeps clock × voltage such that board power stays under the enforced limit (`Enforced Power Limit`, settable with `nvidia-smi -pl`). Hit the cap and DVFS drops the SM clock to fit the power budget. This is *intentional* when you set it (rack/PDU/grid constraint) and shows as `SW Power Cap: Active`. There's also a hardware fast-trip, the **power brake** (`HW Power Brake Slowdown`), asserted by external circuitry (e.g. a PDU/EDPp signal) — that one is the facility telling the GPU to back off *now*.
- **Thermal slowdown.** Two tiers. `SW Thermal Slowdown` engages as the GPU approaches its thermal limit (GPU temp or, on HBM stacks, memory temp) and DVFS steps the clock down to hold temperature. `HW Thermal Slowdown` is the emergency hardware trip near the shutdown threshold — a much harder, deeper clock cut to avoid damage. SW thermal = "cooling is marginal"; HW thermal = "cooling is failing."

The key framing: **a power cap is a decision, a thermal slowdown is a symptom.** If you're power-capped, you chose that trade (or your facility did) and you can reason about it. If you're thermally slowed, you are paying for cooling capacity you don't actually have — the heat isn't leaving fast enough, so the silicon protects itself by delivering fewer FLOPs.

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

You can also query just the mask, which is scriptable for DCGM/Prometheus exporters:

```
$ nvidia-smi --query-gpu=clocks_throttle_reasons.active,clocks.sm,clocks.max.sm,power.draw,enforced.power.limit,temperature.gpu --format=csv
clocks_throttle_reasons.active, clocks.sm [MHz], clocks.max.sm [MHz], power.draw [W], enforced.power.limit [W], temperature.gpu
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

## Worked example

Put a load on the GPU, read the throttle state, then *cause* a throttle and watch the clock move.

Baseline under load (a training step or `dcgmproftester`/`gpu-burn` for a synthetic load):

```
$ nvidia-smi --query-gpu=clocks.sm,power.draw,enforced.power.limit,temperature.gpu,clocks_throttle_reasons.active --format=csv,noheader
1980 MHz, 642.1 W, 700.00 W, 68, 0x0000000000000001
```

`0x1` is `GpuIdle`? No — bit `0x1` is the "GPU Idle" reason only when no work is queued; under a real load with clock at max and draw well under the 700 W cap you'll typically see `0x0` (Not Active). Read this as: clock at boost (1980), power headroom (642 < 700), cool (68 °C) — the GPU is delivering full spec. Good baseline.

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

## Practice

Feeds the **Topology Teardown** deliverable.

1. Put a sustained load on one GPU (real training step, or `gpu-burn` / `dcgmproftester`). Run `nvidia-smi -q -d PERFORMANCE,POWER` and locate the `Clocks Throttle Reasons` block plus current SM clock, max SM clock, power draw, enforced power limit, and temperature.
2. Record the baseline: is any throttle reason `Active`? What's `clocks.sm` vs `clocks.max.sm`?
3. Lower the power cap: `sudo nvidia-smi -pl <below the load's draw>` (e.g. 400 on a 700 W part). Re-read the throttle block and the SM clock. Confirm `SW Power Cap: Active` and record the new clock. If you have a throughput number (samples/sec, tokens/sec), record before/after.
4. Restore: `sudo nvidia-smi -pl 700` (or the card's default). Note whether you ever observed a *thermal* reason at full power — if so, that's cooling debt, not an intentional cap.

**Acceptance:** a note in the Topology Teardown showing at least one throttle reason observed (with its bit/name), the clock delta it caused (SM MHz before/after), and, where available, the throughput impact — plus one sentence classifying it as intentional (power cap) vs symptomatic (thermal). Paste the relevant `nvidia-smi -q` lines.

## Self-check

**(a)** Which throttle reason means "you're paying for cooling you don't have" vs "you capped power on purpose"?

**Answer:** `SW Thermal Slowdown` (and its emergency tier `HW Thermal Slowdown`) means the GPU is clock-limited because it can't shed heat fast enough — you're paying for cooling capacity you don't actually have. `SW Power Cap` (and the facility-driven `HW Power Brake`) means the clock is limited to stay under an enforced power limit that you or the facility set on purpose. Thermal = symptom of a cooling shortfall; power cap = an intentional watts-for-FLOPs trade.

**(b)** Utilization reads 100% but clocks are capped — is the GPU "busy," and what's the cost story?

**Answer:** It's "busy" only in the sense that a kernel is resident every sample window — that's all `GPU-Util` measures. It is *not* delivering full throughput: at a capped/throttled clock it does fewer cycles per second, so tensor-core FLOPs/sec are down proportional to the clock deficit (e.g. 1590 vs 1980 MHz ≈ 20% fewer). The cost story: you pay full GPU rate for a GPU delivering, say, 80% of spec, and utilization hides it. The undelivered 20% is real money — only `clocks.sm` vs `clocks.max.sm` and the throttle reason expose it.

**(c)** Why does a fully populated rack underperform per-GPU spec — name two mechanisms?

**Answer:** (1) **Shared power budget** — the PDU/branch circuit is finite, so a fully populated rack is capped per-GPU/per-node to stay under the breaker (`SW Power Cap`/`HW Power Brake`), holding every GPU below its boost clock. (2) **Shared cooling / thermal coupling** — each GPU's heat raises the local inlet/coolant temperature, so at full density GPUs approach the thermal limit and hit `SW Thermal Slowdown`. Both drop clocks while per-GPU utilization still reads 100%, so aggregate throughput falls short of `8 × single-GPU spec` because watts and cooling, not silicon, are the binding constraint.

## Resources

1. **NVIDIA DGX H100 — Power Capping** — the authoritative doc on setting and reasoning about enforced power limits on H100-class systems. https://docs.nvidia.com/dgx/dgxh100-user-guide/power-capping.html
2. **`nvidia-smi` manual / `-q` device query reference** — the full list of `-d` query groups (`PERFORMANCE`, `POWER`, `CLOCK`) and the `clocks_throttle_reasons.*` fields for scripting. https://docs.nvidia.com/deploy/nvidia-smi/index.html
3. **Gregg, *Systems Performance* (2nd ed.), targeted** — the CPU/thermal DVFS and power-management chapters for the general model of frequency scaling under power/thermal limits; read the GPU as a fast-DVFS instance of the same mechanism. https://www.brendangregg.com/systems-performance-2nd-edition-book.html
