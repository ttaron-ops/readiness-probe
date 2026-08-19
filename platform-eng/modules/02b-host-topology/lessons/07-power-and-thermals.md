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
sources: 11
---
# 02b.7 · Power and thermals as measurable undelivered FLOPs
> **Concept.** A throttled clock is paid-for-but-undelivered compute — power caps and thermal limits force DVFS clock-downs that `nvidia-smi` will name for you, while utilization keeps reading 100%. At fleet scale the same phenomenon shows up a level up, as synchronized power swings that stress the building, not just the die.
>
> Module: [🧬 02b — Host hardware and topology](../README.md) · Deliverable: [Topology Teardown](../practice/topology-teardown/README.md)

## Where this fits

Lesson 06 established that a data path can be topologically correct on paper — right NUMA node, right PCIe switch, GDS enabled — and still under-deliver because the pipe is starved or the fast path silently fell back. You learned to distrust "GDS enabled" until `gdscheck -p` and `/proc/driver/nvidia-fs/stats` confirm it. This lesson applies the same distrust-the-green-light discipline to a different resource: **clock frequency**.

Everything through lesson 06 assumed that once data reaches the GPU, the GPU runs at its rated clock. It very often does not. Power and thermal governors pull the clock below boost, and unlike a misrouted PCIe lane, this cause is *named for you* in a bitmask that almost nobody reads. This lesson closes the last gap before the capstone: after it, you have every host-side cause of "GPU busy, throughput low" in hand — NUMA placement, PCIe link state, storage placement, and now power and thermals — which is exactly the diagnostic tree lesson 08 asks you to walk from symptoms alone. It also gives you the second half of the deliverable's cost story: lesson 06 measured bandwidth you didn't get; this lesson measures FLOPs you didn't get.

## Why this matters

**Utilization is a liar about clocks.** `nvidia-smi` can report `GPU-Util: 100%` while the SM clock sits hundreds of megahertz below its boost ceiling because the GPU hit a power cap or a thermal limit. Those megahertz are FLOPs you are paying for and not receiving, and unlike a data stall the waste is *directly observable and explicitly labelled*: the GPU tells you exactly why it slowed down, in a documented NVML bitmask.

You can put an exact number on it, because tensor throughput is linear in clock. An H100 SXM5 boosts to **1,980 MHz** and is rated at **989.4 TFLOPS dense BF16** on its tensor cores. That is 0.4997 TFLOPS per MHz. Run the same silicon at its **1,590 MHz base clock** under a power cap and you get:

```
  989.4 TFLOPS × (1590 / 1980)  =  794.5 TFLOPS
  deficit = 194.9 TFLOPS = 19.7% of rated throughput
```

Nineteen point seven percent, on an asset renting at roughly $2–3/GPU-hr, rendered by the utilization dashboard as "fully busy." Knowing which throttle reason is *intentional* (you set a power cap for rack or grid reasons) versus a *cooling shortfall* (you are paying for heat removal you do not have) is the difference between a defensible capacity plan and silent burn.

The stakes are not only per-node. Production ML fleets have found that synchronized training-step power ramps across thousands of accelerators can swing facility-level demand by **tens of megawatts, near-instantaneously** — a load profile that damages shared electrical infrastructure (rectifiers, transformers, UPS batteries) independent of any single chip's thermal state. That is a capital-equipment-lifecycle cost, not just a throughput one, and it is exactly the kind of fact a staff-level interviewer at a neocloud or hyperscaler expects you to already know.

## What's new here (calibration)

- **You already know:** CPUs do DVFS, thermal throttling exists as a concept, and `nvidia-smi` shows temperature and power at a glance. You have dealt with CPU turbo, C-states and RAPL on the Linux side, so *frequency scaling under a physical constraint* is not a new idea.
- **New:** the exact NVML **clocks-event-reason bitmask** — eight defined bits with their hex values, what each one means physically, who owns the fix, and the driver-era rename from "Clocks Throttle Reasons" to "Clocks Event Reasons" that breaks naive log greps.
- **New:** the **temperature threshold model** — `nvmlTemperatureThresholds` with its four operational entries (shutdown, slowdown, memory max, GPU max), the concrete H100 values, and the newer `T.Limit` output format that reports *margin to threshold* rather than an absolute temperature.
- **New:** the **power limit model** — min / default / max / requested / enforced, what `nvidia-smi -pl` actually writes, why persistence mode matters, and how DVFS converges on a clock that fits the budget.
- **New:** **node and rack power arithmetic you can re-run** — the full 8-GPU node budget against the DGX H100's published 10.2 kW, three-phase circuit capacity from first principles, and how many nodes a 415 V/32 A N+1 rack actually holds.
- **New:** **cooling arithmetic** — `Q = ṁ·cp·ΔT` worked for both air and liquid, the ~3,500× volumetric-heat-capacity ratio between water and air that explains the industry's move to direct-to-chip liquid, and the ΔT a real 120 kW rack runs at.
- **New:** **synchronized power ramping** at fleet scale, the adjacent **Xid/SXid** fault layer, and the production health-check practice that operationalises all of this continuously.

## Core concepts

### 1. The physics: why clock is the lever

Dynamic power in CMOS is, to first order:

```
  P_dynamic  =  α · C · V² · f          + P_static(V, T)
                │   │   │    │             │
                │   │   │    │             └─ leakage: rises with voltage AND temperature
                │   │   │    └─ switching frequency
                │   │   └─ supply voltage — SQUARED
                │   └─ switched capacitance (fixed by the design)
                └─ activity factor (what fraction of transistors toggle per cycle)
```

Three consequences drive everything in this lesson:

1. **Voltage is quadratic, frequency is linear** — but they are *coupled*: a higher clock needs a higher voltage to meet timing. So the practical power-versus-clock curve is closer to cubic than linear, which is why a modest clock reduction buys a large power reduction. Dropping from 1,980 to 1,590 MHz (a 20% clock cut) with the accompanying voltage reduction can cut board power by 35–45%. That asymmetry is why power capping is an effective lever and why the throughput you lose is smaller than the watts you save.
2. **Leakage rises with temperature**, so a hot chip burns more power at the same clock, which makes it hotter. Without a governor that is a positive feedback loop; the governor is what breaks it.
3. **The activity factor α is set by your workload.** A dense GEMM saturating the tensor cores has a far higher α than a memory-bound kernel. This is why the *same* GPU at the *same* clock draws wildly different power on different kernels, and why "our GPUs never hit the power cap" is a statement about your workload mix, not about your hardware.

**DVFS** (Dynamic Voltage and Frequency Scaling) is the control loop that picks a (V, f) operating point every few milliseconds so the chip stays inside its constraints. On an NVIDIA data-center GPU the constraints are, roughly in order of how fast they act:

| Constraint | Sensor | Response time | What it protects |
|---|---|---|---|
| Board power vs enforced limit | on-board power telemetry | ~ms (software/firmware loop) | the power delivery network and your PDU budget |
| External power-brake assertion | a hardware signal pin from the PSU/system | µs (hardware fast-trip) | the power supply from an over-current event |
| GPU die temperature vs max-operating | on-die thermal diodes | ~ms | sustained reliability |
| HBM stack temperature vs max-operating | in-stack sensors | ~ms | memory retention and refresh integrity |
| GPU die temperature vs slowdown threshold | on-die thermal diodes | µs (hardware) | the silicon, right now |
| GPU die temperature vs shutdown threshold | on-die thermal diodes | µs (hardware) | the silicon, absolutely |

**None of these are bugs.** Every one is the chip trading clock for staying inside a physical envelope, working exactly as designed. The platform-engineering question is never "why is this broken" — it is **"which constraint is binding, and did I choose it?"**

### 2. The clocks-event-reason bitmask, exactly

NVML exposes the active constraint set as a 64-bit mask via `nvmlDeviceGetCurrentClocksEventReasons()`. The defined bits, verbatim from `nvml.h`:

| Bit | Constant | Hex | What it means | Who owns the fix |
|---|---|---|---|---|
| — | `nvmlClocksEventReasonNone` | `0x0` | Clocks are as high as possible. Nothing is limiting. | — |
| 0 | `nvmlClocksEventReasonGpuIdle` | `0x1` | Nothing is running; clocks dropped to idle. Not a problem — but if you see it during what you believe is a training run, your job is not on this GPU. | you |
| 1 | `nvmlClocksEventReasonApplicationsClocksSetting` | `0x2` | Clocks pinned to application-specified values (`nvidia-smi -ac`). Deprecated on modern parts. | you |
| 2 | `nvmlClocksEventReasonSwPowerCap` | `0x4` | **Clocks were reduced to stay under the currently set power limit.** The single most common bit you will see on a busy data-center GPU. | you / facilities — it is a *decision* |
| 3 | `nvmlClocksThrottleReasonHwSlowdown` | `0x8` | **Hardware slowdown is engaged** — a roll-up bit. NVML's own documentation lists three possible causes: temperature too high, an external power-brake assertion (e.g. from the system power supply), or power draw too high with fast-trigger protection reducing clocks. May also appear transiently during a P-state or clock change. | depends — decompose it with bits 6 and 7 |
| 4 | `nvmlClocksEventReasonSyncBoost` | `0x10` | This GPU is in a **sync boost group** and is being held to the minimum clock achievable across the whole group. Standard on DGX/HGX so all eight GPUs step together. **The interesting consequence: this GPU's clock is low because of a *different* GPU's constraint** — go read the other GPUs' reasons. | look at the peers |
| 5 | `nvmlClocksEventReasonSwThermalSlowdown` | `0x20` | **Software thermal slowdown**: clocks reduced so that GPU temperature stays under GPU Max Operating Temperature *and* memory temperature stays under Memory Max Operating Temperature. Cooling is marginal. | facilities — it is a *symptom* |
| 6 | `nvmlClocksThrottleReasonHwThermalSlowdown` | `0x40` | **Hardware thermal slowdown — core clocks reduced by a factor of 2 or more.** This is the emergency tier. Cooling is failing. | page someone |
| 7 | `nvmlClocksThrottleReasonHwPowerBrakeSlowdown` | `0x80` | **External power-brake assertion** triggered — the system power supply told the GPU to back off *now*, and clocks were cut by 2× or more. | facilities / power design |
| 8 | `nvmlClocksEventReasonDisplayClockSetting` | `0x100` | Clocks limited by display clock settings. Irrelevant on headless data-center parts. | — |

**Two naming traps.**

First, NVIDIA **renamed the whole group**: `nvmlClocksThrottleReason*` became `nvmlClocksEventReason*`, and the old names are now `#define`d aliases marked deprecated. `nvidia-smi` followed: newer drivers print the section header as **`Clocks Event Reasons`** where older ones printed **`Clocks Throttle Reasons`**, and the CSV query field is `clocks_event_reasons.active` with `clocks_throttle_reasons.active` retained as a deprecated alias. If you have an exporter or a log grep matching the old string, it will silently stop matching after a driver upgrade. Match on both, or on the numeric field.

Second — and this is the one that produces wrong diagnoses — **`HwSlowdown` (0x8) is a roll-up, not a cause.** Seeing `0x8` alone tells you the hardware fast path engaged but not why. Decompose it:

```
  active & 0x40  →  HwThermalSlowdown      : it is heat
  active & 0x80  →  HwPowerBrakeSlowdown   : it is the power supply
  active & 0x8 only, neither of the above  →  fast-trigger power protection,
                                              or a transient during a clock change
```

**The framing that matters:** *a power cap is a decision; a thermal slowdown is a symptom.* If you are power-capped, you (or your facility) chose that trade and you can reason about it — watts for FLOPs, deliberately. If you are thermally slowed, you are paying for cooling capacity you do not actually have: the heat is not leaving fast enough, so the silicon protects itself by delivering fewer FLOPs, and nobody made that decision on purpose.

### 3. The temperature threshold model

NVML defines the thresholds via `nvmlTemperatureThresholds_enum`. The four that matter operationally:

| Enum | Index | Meaning |
|---|---|---|
| `NVML_TEMPERATURE_THRESHOLD_GPU_MAX` | 3 | **GPU Max Operating Temperature** — above this the GPU can be throttled below base clock. This is what `SW Thermal Slowdown` defends. |
| `NVML_TEMPERATURE_THRESHOLD_MEM_MAX` | 2 | **Memory Max Operating Temperature** — the HBM equivalent, and it defends against a *software* slowdown too. On HBM3 parts this is frequently the binding constraint before the die is. |
| `NVML_TEMPERATURE_THRESHOLD_SLOWDOWN` | 1 | Temperature at which the GPU begins **hardware** slowdown. |
| `NVML_TEMPERATURE_THRESHOLD_SHUTDOWN` | 0 | Temperature at which the GPU shuts down for hardware protection. |

For the **H100 PCIe** part, NVIDIA's product brief gives concrete numbers you should know by heart because they are the reference points for every H100-generation reading:

```
  GPU Max Operating Temperature (software throttle begins)   87 °C
  GPU Slowdown Temperature      (hardware slowdown begins)   89 °C
  GPU Shutdown Temperature      (hardware protection)        92 °C
  HBM thermal qualification temperature                      95 °C
```

Only **five degrees** separate "start giving back clock" from "hardware cuts you in half," and only three more from a hard shutdown. That narrow band is why a cooling problem escalates from a throughput annoyance to an outage so quickly, and why an 88 °C alert threshold — the figure Modal's fleet-operations write-up cites as a practical operational red flag — is a sane place to page before scheduling new work onto a node.

**A format change you will trip over.** Newer drivers report thresholds as **`T.Limit`** values: the *margin remaining* to the threshold, in degrees, rather than the absolute temperature. So `GPU T.Limit Temp : 15 C` means "15 °C of headroom left," and the number **falls** as the GPU heats up, reaching 0 at the limit. Older tooling that alerts on "temperature above X" will read a T.Limit field as a suspiciously cool GPU. Check which format your driver emits before wiring an alert:

```
$ nvidia-smi -q -d TEMPERATURE | sed -n '/Temperature/,/^$/p'
    Temperature
        GPU Current Temp                  : 71 C
        GPU T.Limit Temp                  : 16 C      ← margin, not absolute
        GPU Shutdown T.Limit Temp         : -5 C      ← relative to the same reference
        GPU Slowdown T.Limit Temp         : -2 C
        GPU Max Operating T.Limit Temp    : 0 C
        GPU Target Temperature            : N/A
        Memory Current Temp               : 78 C
        Memory Max Operating T.Limit Temp : 0 C
```

Read that block: the reference point is "GPU Max Operating," expressed as `0 C`. Slowdown is `-2 C` (2 °C *past* max-operating) and shutdown is `-5 C`. Current margin is `16 C`, so this GPU is 16 °C below the point where software thermal slowdown begins. **Memory Current Temp of 78 °C against its own max-operating reference is the number people forget to chart** — on HBM3 parts the memory stack frequently hits its limit before the die does, and the resulting `SW Thermal Slowdown` looks identical in the bitmask.

### 4. The power limit model

`nvidia-smi -q -d POWER` exposes a small hierarchy, and the distinctions matter:

```
$ nvidia-smi -q -d POWER
    GPU Power Readings
        Power Draw                        : 698.44 W    ← instantaneous board power
        Current Power Limit               : 700.00 W    ← what has been requested
        Requested Power Limit             : 700.00 W
        Default Power Limit               : 700.00 W    ← the SKU's factory setting
        Min Power Limit                   : 200.00 W    ← floor you may set
        Max Power Limit                   : 700.00 W    ← ceiling you may set
    Power Samples
        Duration                          : 1.99 sec
        Number of Samples                 : 119
        Max                               : 700.11 W
        Min                               : 642.20 W
        Avg                               : 691.35 W
    Module Power Readings
        Power Draw                        : 731.02 W    ← whole SXM module, incl. HBM+VRM
        Current Power Limit               : N/A
```

| Field | Meaning | Gotcha |
|---|---|---|
| `Power Draw` | Board power, averaged over a short window | Instantaneous samples are noisier than this; `Power Samples` shows the spread. |
| `Default Power Limit` | The SKU's factory limit — 700 W for H100 SXM5, ~350 W for H100 PCIe, ~400 W for H100 NVL | This, not the observed draw, is what you budget a rack against. |
| `Min` / `Max Power Limit` | The settable range. `nvidia-smi -pl <W>` outside it is rejected. | A `Max` *below* `Default` means someone already applied a vBIOS or OOB constraint. |
| `Requested` vs `Enforced/Current` | What you asked for versus what is in force | They diverge when an out-of-band controller (BMC, chassis manager) is also setting a limit. **The lower one wins, and it is not always yours.** |
| `Module Power Readings` | Whole-module power on SXM parts (die + HBM + on-module VRMs) | Consistently higher than board `Power Draw`. Budget racks against *module* power on SXM systems. |

**Setting a limit:**

```
$ sudo nvidia-smi -pm 1                     # persistence mode ON — first, always
Enabled persistence mode for GPU 00000000:9D:00.0.

$ sudo nvidia-smi -i 0 -pl 500
Power limit for GPU 00000000:9D:00.0 was set to 500.00 W from 700.00 W.
```

**Persistence mode matters and is easy to miss.** Without it, the driver unloads when the last client exits, and every setting you applied — power limit, clocks, ECC mode — reverts to default. A power cap applied for PDU-protection reasons that silently reverts between jobs is worse than no cap at all, because your capacity plan assumes it is there. On modern drivers prefer the `nvidia-persistenced` daemon over the deprecated `-pm 1` flag; check which your distribution ships.

Note also that the power limit is **per-GPU and node-local**. Nothing in Kubernetes knows about it. A pod scheduled onto a node whose GPUs are capped at 400 W gets 400 W-worth of throughput and there is no `Allocatable` field that says so — which is precisely the kind of invisible capacity variance that makes fleet-wide benchmarks look noisy.

### 5. Reading it all in one pass, and what each line proves

The four query groups that matter, together:

```
$ nvidia-smi -q -i 0 -d POWER,TEMPERATURE,CLOCK,PERFORMANCE
```

```
    Clocks Event Reasons
        Idle                              : Not Active
        Applications Clocks Setting       : Not Active
        SW Power Cap                      : Active            ← 0x4
        HW Slowdown                       : Not Active        ← 0x8 roll-up
            HW Thermal Slowdown           : Not Active        ← 0x40
            HW Power Brake Slowdown       : Not Active        ← 0x80
        Sync Boost                        : Not Active        ← 0x10
        SW Thermal Slowdown               : Not Active        ← 0x20
        Display Clock Setting             : Not Active
    GPU Power Readings
        Power Draw                        : 699.81 W
        Current Power Limit               : 700.00 W
    Temperature
        GPU Current Temp                  : 71 C
        Memory Current Temp               : 78 C
    Clocks
        Graphics                          : 1590 MHz
        SM                                : 1590 MHz
        Memory                            : 2619 MHz
        Video                             : 1395 MHz
    Max Clocks
        Graphics                          : 1980 MHz
        SM                                : 1980 MHz
        Memory                            : 2619 MHz
```

Read it as a proof chain, not a list of facts:

1. `SW Power Cap : Active` and nothing else → **exactly one constraint is binding, and it is the power limit.**
2. `Power Draw 699.81 W` against `Current Power Limit 700.00 W` → the GPU is pinned *at* the cap, which corroborates (1). If draw were 640 W with the cap bit active, something else would be going on.
3. `SM 1590 MHz` against `Max SM 1980 MHz` → a 390 MHz deficit, **19.7%** of the clock.
4. Both thermal bits inactive and 71 °C die / 78 °C memory → cooling has headroom. **This is not a cooling problem.**
5. Therefore: the GPU is delivering ~794 TFLOPS BF16 instead of ~989, because 700 W is not enough power for this workload's activity factor at boost clock. That is a **hardware/SKU envelope statement**, not a fault, and the only levers are a higher power limit (if the SKU and the rack allow it) or accepting the trade.

The same in one scriptable line, for exporters:

```
$ nvidia-smi --query-gpu=index,clocks.sm,clocks.max.sm,power.draw,enforced.power.limit,\
temperature.gpu,temperature.memory,clocks_event_reasons.active --format=csv,noheader,nounits
0, 1590, 1980, 699.81, 700.00, 71, 78, 0x0000000000000004
```

`0x4` = `SwPowerCap`. If your driver rejects `clocks_event_reasons.active`, fall back to `clocks_throttle_reasons.active` — same field, older name.

**The DCGM field IDs**, for the exporter you will design in lesson 08:

| Field ID | Name | What it gives you |
|---|---|---|
| 100 | `DCGM_FI_DEV_SM_CLOCK` | Current SM clock, MHz |
| 112 | `DCGM_FI_DEV_CLOCKS_EVENT_REASONS` | The bitmask (alias: `DCGM_FI_DEV_CLOCK_THROTTLE_REASONS`) |
| 140 | `DCGM_FI_DEV_MEMORY_TEMP_CELSIUS` | HBM temperature |
| 150 | `DCGM_FI_DEV_GPU_TEMP_CELSIUS` | Die temperature |
| 151 | `DCGM_FI_DEV_MEMORY_MAX_OP_TEMP_CELSIUS` | Memory max-operating threshold |
| 152 | `DCGM_FI_DEV_GPU_MAX_OP_TEMP_CELSIUS` | GPU max-operating threshold |
| 153 | `DCGM_FI_DEV_GPU_TEMP_MARGIN_CELSIUS` | **Thermal margin — degrees to the nearest slowdown threshold.** The single best field to alert on, because it is already normalised across SKUs. |
| 155 | `DCGM_FI_DEV_BOARD_POWER_WATTS` | Board power (alias `DCGM_FI_DEV_POWER_USAGE`) |
| 156 | `DCGM_FI_DEV_TOTAL_ENERGY_CONSUMPTION` | Cumulative energy in **mJ** since driver load — integrate this for real energy accounting rather than sampling power |
| 157 | `DCGM_FI_DEV_BOARD_POWER_RAW_WATTS` | Instantaneous (unaveraged) power |
| 158 / 159 | `..._GPU_TEMP_SLOWDOWN_CELSIUS` / `..._SHUTDOWN_CELSIUS` | The hardware thresholds |
| 160–164 | `..._BOARD_POWER_LIMIT_{REQUESTED,MIN,MAX,DEFAULT,ENFORCED}_WATTS` | The full power-limit hierarchy from §4 |

Field **153** (`GPU_TEMP_MARGIN`) is the one to build your alert on. Absolute temperature thresholds are SKU-specific and go stale every hardware generation; margin-to-threshold is comparable across a mixed fleet and needs no per-SKU table.

### 6. A clock-versus-constraint timeline

The steady-state readings above hide the dynamics. What actually happens when you load a GPU and then squeeze it:

```
   ONE GPU, ONE SUSTAINED GEMM LOAD, TWO CONSTRAINTS ENGAGING IN TURN
   (representative shape; absolute values depend on SKU, cooling and kernel)

 SM clock
 (MHz)
  2000 ┤ ●●●●●●●●
       │         ●●●●
  1900 ┤             ●●●
       │                ●●●●
  1800 ┤                    ●●●●●
       │                         ●●●●●●
  1700 ┤                               ●●●●●●●●
       │                                       ●●●●●●●●●●●
  1600 ┤                                                  ●●●●●●●●●●●●●●●●●
       │                                                                   ●●●●●●●
  1500 ┤                                                                          ●●●●
       └──┬──────┬──────┬──────┬──────┬──────┬──────┬──────┬──────┬──────┬──────┬───▶ t
          0     10s    20s    30s    40s    50s    60s    70s    80s    90s   100s

 Power  ┤                    ┌────────────── pinned at the 700 W enforced limit ──────
 (W)    │            ┌───────┘
   700  ┤     ┌──────┘
   600  ┤ ────┘
        └──────────────────────────────────────────────────────────────────────────▶

 Die    ┤                                              ┌──────── 84 ── 86 ── 87 °C ───
 temp   │                              ┌───────────────┘
   80   ┤              ┌───────────────┘
   70   ┤ ─────────────┘
        └──────────────────────────────────────────────────────────────────────────▶

 event  │ 0x0    │ 0x0  │        0x4 (SW Power Cap)         │ 0x4|0x20 (+SW Thermal) │
 mask   └────────┴──────┴───────────────────────────────────┴────────────────────────┘

  ── PHASE 1 (0-8s): ramp ──────────────────────────────────────────────────────────
     Kernel starts, activity factor α jumps. Clock at boost 1980 MHz. Power climbing
     from idle. Die still cool. mask = 0x0 — nothing limiting. This is the only phase
     where the GPU delivers its rated 989 TFLOPS.

  ── PHASE 2 (8-45s): POWER-limited ────────────────────────────────────────────────
     Power reaches the 700 W enforced limit. DVFS steps (V, f) down until board power
     fits the budget. Clock settles around 1700-1750 MHz. mask = 0x4.
     ▸ This is a DECISION. You chose 700 W (or the SKU did). Cost is knowable and
       chosen: throughput scales as clock/1980.

  ── PHASE 3 (45-100s): also THERMALLY limited ─────────────────────────────────────
     The heat generated in phase 2 has had time to soak through the die → TIM →
     heatsink/cold plate → coolant. Die temperature approaches GPU Max Operating
     (87 °C on H100). SW Thermal Slowdown engages and pulls the clock down FURTHER,
     below what the power budget alone required. mask = 0x4 | 0x20 = 0x24.
     ▸ This is a SYMPTOM. Nobody chose it. You are paying for cooling you don't have.

  ── THE TELL ──────────────────────────────────────────────────────────────────────
     A 30-second benchmark stops at the end of phase 2 and reports a clock that
     never falls. A 20-minute production job lives in phase 3. THAT is why acceptance
     tests must run long enough to reach thermal steady state — typically 10-15
     minutes, not 60 seconds. Watching clocks.sm converge is how you know you got there.
```

**The operational rule that falls out: never trust a short benchmark for thermal characterisation.** The power constraint binds in seconds; the thermal constraint binds in minutes, because it depends on the heat capacity of everything between the die and the facility loop.

### 7. The node power budget, computed

Now scale from one die to one chassis. The anchor is a published number: **NVIDIA's DGX H100 datasheet states a maximum system power consumption of 10.2 kW**, delivered by **six 3,300 W Titanium-rated power supplies in a 4+2 redundant arrangement**, each rated 200–240 V, 16 A, 50–60 Hz, with four PSUs required to be energised for the system to run.

Build the budget bottom-up and see what the GPUs actually account for:

| Component | Count | Each | Subtotal | Provenance |
|---|---|---|---|---|
| H100 SXM5 GPU | 8 | 700 W | **5,600 W** | NVIDIA SKU TDP — hard number |
| Intel Xeon Platinum 8480C | 2 | 350 W | **700 W** | Intel SKU TDP — hard number |
| DDR5 RDIMM (2 TB total, 32 × 64 GB) | 32 | ~7 W | **~224 W** | typical loaded RDIMM power — estimate |
| U.2 NVMe (data cache) | 8 | ~20 W | **~160 W** | typical enterprise U.2 active power — estimate |
| M.2 NVMe (OS, RAID 1) | 2 | ~8 W | **~16 W** | estimate |
| ConnectX-7 400G + OSFP optics | 8 | ~27 W | **~216 W** | optics alone are 8–9 W max at 400G per NVIDIA's transceiver spec; card+optics estimate |
| Additional NICs / DPUs, in-band + storage | 2–4 | ~30 W | **~90 W** | estimate |
| **Subtotal, itemised** | | | **~7,006 W** | |
| **Residual to the 10.2 kW datasheet figure** | | | **~3,194 W** | 4 × NVSwitch, PCIe switches and retimers, baseboard VRM conversion losses, fans, PSU conversion loss |

Three things to take from that table, in order of importance:

1. **The GPUs are 55% of the node, not 90%.** The single most common capacity-planning error is budgeting `8 × 700 W = 5.6 kW` and being surprised when the rack trips. **The non-GPU 45% is real and it is mostly not itemised on any datasheet.**
2. **Fans and VRM/PSU conversion losses are a large, invisible slice.** At ~10 kW in a compact chassis, moving enough air is itself a kilowatt-class problem (see §8), and every conversion stage — PSU AC→12 V, board 12 V→VRM, VRM→core voltage — sheds a few percent. Titanium-rated PSUs are ~96% efficient at 50% load; that alone is ~400 W of heat at 10 kW.
3. **Newer generations move the anchor, not the method.** A B200 SXM part is a 1,000 W class GPU and an 8-GPU DGX B200 draws on the order of 14 kW; a GB200 NVL72 rack is roughly 120 kW for 72 GPUs. Re-run the same arithmetic with the new anchor rather than scaling the old answer.

### 8. Rack power: from a breaker rating to a node count

Three-phase power, with the derivation you should be able to do on a whiteboard:

```
  S (apparent power, VA)  =  √3 × V_line-to-line × I
  P (real power, W)       =  S × PF

  For the NVIDIA DGX SuperPOD H100 reference: 415 V three-phase Wye
  (415 V line-to-line, 240 V line-to-neutral), 32 A breaker:

    S = 1.732 × 415 V × 32 A                       = 23,010 VA  ≈ 23.0 kVA
    equivalently: 3 × 240 V × 32 A                 = 23,040 VA   (same thing)
    P = 23.0 kVA × PF 0.95 (typical server PSU)    ≈ 21.8 kW per circuit
```

That 21.8 kW figure matches the value NVIDIA publishes in the DGX SuperPOD H100 data-center design guide for exactly this configuration — a useful cross-check that the arithmetic is right.

Now the rack:

```
  Reference design: 4 × DGX H100 per rack, three rPDUs on independent
  circuits (A, B, C), N+1, and NO circuit may carry more than 50% of the load.

  IT load          = 4 nodes × 10.2 kW              = 40.8 kW
  Circuits         = 3 × 21.8 kW                    = 65.4 kW installed
  Surviving one circuit failure = 2 × 21.8 kW       = 43.6 kW
  Check: 43.6 kW ≥ 40.8 kW                          ✓  (93.6% loaded on N-1)
  Per-circuit share in normal operation = 40.8 / 3  = 13.6 kW = 62% of 21.8 kW
```

The "no circuit above 50%" rule is what forces three circuits rather than two: with only two, losing one would leave the survivor carrying 100% of a 40.8 kW load against a 21.8 kW capacity — an immediate trip and a whole-rack outage.

**Where the power cap comes in.** Suppose facilities can only give you two circuits. Then your survivable budget is 21.8 kW, and four nodes at 10.2 kW do not fit. Options:

```
  (a) 2 nodes per rack             → 20.4 kW ≤ 21.8 kW ✓ but you halve rack density
  (b) 4 nodes, GPUs power-capped   → need node power ≤ 21.8/4 = 5.45 kW.
                                     Non-GPU load is ~4.6 kW, so GPU budget = 0.85 kW
                                     for eight GPUs = 106 W each.  ABSURD — this
                                     configuration simply does not exist.
  (c) 3 nodes, GPUs capped         → node budget 7.27 kW; GPU budget 2.67 kW / 8
                                     = 334 W per GPU. Below the H100 SXM Min Power
                                     Limit of ~200 W? No — but at 334 W of a 700 W
                                     part you are running near base clock or below,
                                     giving up ~25-35% of throughput permanently.
```

Working (b) and (c) is the point of the exercise: **once the non-GPU 45% is fixed, power capping has far less room than intuition suggests.** You cannot cap your way out of a badly provisioned rack; you can only trade a large amount of throughput for a small amount of headroom. That is the argument to bring to a facilities conversation, with numbers.

### 9. Cooling: the same energy, on the other side of the die

Every watt that goes in comes out as heat. The transport equation is the same for air and for water:

```
  Q  =  ṁ · c_p · ΔT          (mass flow form)
  Q  =  V̇ · ρ · c_p · ΔT      (volumetric flow form)

  where  Q  = heat removed (W)
         V̇  = volumetric flow rate (m³/s)
         ρ  = density (kg/m³)
         c_p = specific heat capacity (J/(kg·K))
         ΔT = temperature rise of the coolant across the load (K)
```

The two fluids, at ~20 °C:

| Fluid | ρ (kg/m³) | c_p (J/(kg·K)) | **ρ·c_p** (J/(m³·K)) |
|---|---|---|---|
| Air | 1.20 | 1,005 | **1,206** |
| Water | 1,000 | 4,180 | **4,180,000** |

**Water carries ~3,470× more heat per unit volume than air at the same ΔT.** That single ratio is the entire engineering argument for direct-to-chip liquid cooling, and it is worth being able to derive rather than assert.

**Air, worked for one DGX H100 node:**

```
  Q  = 10,200 W
  ΔT = 15 K   (a typical, achievable inlet-to-exhaust rise)

  V̇ = Q / (ρ·c_p·ΔT) = 10,200 / (1,206 × 15) = 0.564 m³/s

  in CFM (1 m³/s = 2,118.9 CFM):  0.564 × 2118.9 = 1,195 CFM per node
  for a 4-node, 40.8 kW rack:                       4,780 CFM
```

~1,200 CFM through an 8U chassis is loud, and it is *achievable* — DGX H100 is an air-cooled system, and its operating inlet range is a narrow **5–30 °C**. Note what happens if the inlet is warm: you have less ΔT budget before the exhaust — and therefore the die — reaches the throttle point, so the same airflow removes less heat. **A hot aisle raises the throttle rate on the entire rack, and it shows up as `SW Thermal Slowdown` on the top-of-rack nodes first.**

**Air, worked for a 120 kW rack** — the density a GB200 NVL72 runs at:

```
  V̇ = 120,000 / (1,206 × 15) = 6.63 m³/s = 14,050 CFM through one rack
```

Fourteen thousand CFM into a 600 mm-wide cabinet is not a cooling design, it is a wind tunnel. This is where air runs out, and it is why the GB200 NVL72 is liquid-cooled by mandate rather than by preference.

**Liquid, worked for the same 120 kW rack:**

```
  Q(kW) = flow(L/min)/60 × ρ(1 kg/L) × c_p(4.18 kJ/(kg·K)) × ΔT

  Published GB200 NVL72 coolant flow: up to ~130 L/min per rack.

  ΔT = Q × 60 / (c_p × flow)
     = 120 × 60 / (4.18 × 130)
     = 7,200 / 543.4
     = 13.2 K
```

A 13 K rise across the rack, at 130 L/min, removes 120 kW. Compare the plumbing: a 130 L/min pipe is roughly a 1-inch line, versus 14,000 CFM of air. NVIDIA's guidance for GB200-class racks is a facility water inlet in the **20–25 °C** band; cooling-solution vendors publish higher design maxima (a QCT stand-alone CDU specification, for example, cites a 45 °C maximum liquid inlet and 65 °C maximum return) because a CDU sits between the secondary (server) loop and the facility loop and can run the two at different temperatures.

Rearranged, the same equation is the design tool: **for a fixed heat load, flow rate and ΔT trade off directly.** Halve the flow and you double the ΔT, which raises the cold-plate temperature, which raises the die temperature, which brings you closer to `SW Thermal Slowdown`. That is the whole causal chain from "the CDU pump is running at reduced speed" to "throughput is down 8%" — and it is invisible in every GPU metric except the thermal margin.

```
   THE THERMAL PATH — WHERE EACH ΔT IS SPENT, AND WHO OWNS IT

  ┌───────────────────────────────────────────────────────────────────────────┐
  │ ① GPU DIE (junction)                              T_j  ~ 75-87 °C         │
  │    700 W generated in ~814 mm² of silicon                                 │
  │    owner: the workload's activity factor α                                │
  └───────────────────────┬───────────────────────────────────────────────────┘
                          │  ΔT ≈ 5-15 K   across the thermal interface material
                          │                (TIM degrades with age → this ΔT GROWS
                          │                 over a node's life; a 3-year-old node
                          │                 throttles earlier than a new one)
                          ▼
  ┌───────────────────────────────────────────────────────────────────────────┐
  │ ② COLD PLATE base  (or heatsink base, on air-cooled parts)                │
  │    owner: the OEM's mechanical design + how well it was torqued at build  │
  └───────────────────────┬───────────────────────────────────────────────────┘
                          │  ΔT ≈ 5-10 K   plate-to-coolant convective resistance
                          │                (falls as flow rate rises)
                          ▼
  ┌───────────────────────────────────────────────────────────────────────────┐
  │ ③ SECONDARY LOOP  (server-side coolant, a water/glycol mix)               │
  │    supply ~30 °C  ──▶  return ~43 °C     ΔT = 13 K across the rack        │
  │    Q = flow/60 × 4.18 × ΔT     ← the equation above, this is where it lives│
  │    owner: the CDU pump curve and the manifold/quick-disconnect design      │
  └───────────────────────┬───────────────────────────────────────────────────┘
                          │  heat exchanger inside the CDU
                          │  ΔT ≈ 2-5 K approach temperature
                          ▼
  ┌───────────────────────────────────────────────────────────────────────────┐
  │ ④ CDU  (Coolant Distribution Unit)  — isolates server loop from facility  │
  │    owner: the datacenter operator                                          │
  └───────────────────────┬───────────────────────────────────────────────────┘
                          │
                          ▼
  ┌───────────────────────────────────────────────────────────────────────────┐
  │ ⑤ FACILITY WATER LOOP  ──▶  cooling towers / chillers  ──▶  outside air   │
  │    owner: the building. Ultimately limited by WET-BULB TEMPERATURE.        │
  │    On a hot, humid day the facility loop runs warmer, every ΔT above it    │
  │    shifts up by the same amount, and T_j rises → SW Thermal Slowdown.      │
  └───────────────────────────────────────────────────────────────────────────┘

  ── THE KEY INSIGHT ────────────────────────────────────────────────────────
  T_junction = T_facility_water + Σ(all the ΔT stages above)

  Every stage is a series resistance. A degraded pump (③), a fouled heat
  exchanger (④), aged TIM (①), or a 35 °C afternoon (⑤) each add a few K, and
  they ADD. The GPU cannot tell them apart — it only sees T_j crossing 87 °C
  and sets bit 0x20. Which is why "SW Thermal Slowdown active" is a
  FACILITIES ticket that starts at ⑤ and works upward, not a GPU ticket.

  ── AIR-COOLED VARIANT ─────────────────────────────────────────────────────
  ① die → TIM → ② heatsink fins → ③ chassis airflow (1,195 CFM @ ΔT 15 K
  for a 10.2 kW node) → hot aisle → CRAC/CRAH → facility loop. Same series-
  resistance logic, but stage ③ has 3,470× less volumetric heat capacity,
  so its ΔT is far more sensitive to flow — one failed fan is worth several K.
```

### 10. Why a fully populated rack underperforms `N × single-GPU spec`

Two mechanisms, and they compound:

1. **Shared power budget.** From §8, the circuit capacity is fixed and the N-1 survivability rule consumes a large fraction of it. If aggregate draw would approach the limit, operators apply per-node or per-GPU power caps — so every GPU runs with `SW Power Cap` active, below boost, all the time. The rack is oversubscribed on watts and no single GPU gets its full envelope. This one at least shows up honestly in the bitmask.
2. **Shared cooling and thermal coupling.** From §9, `T_junction = T_facility + Σ ΔT`, and a full rack raises the common terms for everyone: higher aggregate heat means a warmer return, which means a warmer supply for the next node, which means every die runs hotter at the same power. `SW Thermal Slowdown` appears first on the GPUs with the worst position in that chain — typically the middle of a chassis (surrounded by heat on both sides) and the top of a rack (fed by air that has already picked up heat). **Each GPU's heat is its neighbour's cooling problem, so density itself lowers the achievable clock.**

Both are invisible in per-GPU utilization: every GPU reads "busy," and the rack delivers less than `8 × single-GPU spec` because watts and cooling — not silicon — are the binding constraints. That gap is the cost story, and the two bits (`0x4` versus `0x20`) tell you which mechanism you are looking at.

### 11. Beyond one GPU: synchronized fleet-wide power ramping

Everything above is steady-state: one GPU, sustained load, a clock that settles. Production ML fleets have surfaced a second, distinct phenomenon that only appears at scale.

Training workloads have a **step structure**: compute-heavy phases (forward and backward passes, high α, high power) alternate with comparatively idle phases (gradient all-reduce, optimizer step, data-loader gaps, low α, low power). Because a data-parallel or tensor-parallel job executes step boundaries in near-lockstep across every rank — that is what synchronous distributed training *is* — thousands of chips ramp from low power to peak and back **almost simultaneously**.

Google Cloud's engineering team documented this directly: synchronized ML power draw can swing a cluster's electrical demand by **tens of megawatts, near-instantaneously** — a load profile unlike traditional, uncorrelated datacenter workloads, and one that stresses rectifiers, transformers and UPS batteries that were not designed for that rate of change. Their mitigation was compiler-level: detect the workload's power "signature" and reshape *when* compute blocks run within a step so the aggregate power curve is smoothed, instead of letting every chip ramp in lockstep. Measured results: **roughly 50% reduction in cluster-level power fluctuation**, **per-chip hotspot temperature swings reduced from roughly 20 °C to roughly 10 °C**, at **under 1% performance overhead**. That work is TPU-specific, but the mechanism — smoothing compute-block activity around synchronization points — is a general, transferable idea rather than an artifact of one vendor's silicon.

The reframe this earns you: **`nvidia-smi -q` on one GPU can never show you this.** It is an aggregate fact about *when* thousands of chips draw power together, not about any single chip's steady-state clock. Every individual GPU can respect its `enforced.power.limit` at every sample and the rack PDU still sees a repeating multi-kilowatt step function for the whole run. A staff engineer who only checks per-GPU throttle reasons has covered the die-level story and missed the facility-level one.

There is a per-node echo of this you *can* see: watch the second-derivative of `DCGM_FI_DEV_BOARD_POWER_RAW_WATTS` (field 157, instantaneous rather than averaged) against step boundaries. The averaged field 155 smooths exactly the transient you are looking for.

### 12. The adjacent diagnostic layer: Xid and SXid

Throttling and **Xid errors** are complementary but distinct signals, and conflating them routes you to the wrong remediation.

- A **throttle reason** is DVFS working correctly — a policy response to a physical constraint. The GPU is healthy. The fix is a power-cap or cooling conversation.
- An **Xid error**, surfaced in `dmesg` by the NVIDIA driver, is a hardware or driver **fault** event: something actually misbehaved. The classic critical example is **Xid 79, "GPU has fallen off the bus"** — the driver can no longer communicate with the device over PCIe. Thermal stress and power fluctuation in dense racks are among its triggers, but it is categorically not the same thing as a throttle bit being set. **SXid** is the NVSwitch-specific equivalent, relevant on multi-GPU NVSwitch systems where the switch fabric rather than a GPU is the faulting component.

```
$ dmesg -T | grep -iE 'xid|sxid'
[Sun Aug 17 04:12:07 2026] NVRM: Xid (PCI:0000:9d:00): 79, pid=..., GPU has fallen off the bus.
[Sun Aug 17 04:12:07 2026] NVRM: GPU 0000:9d:00.0: GPU serial number ...
```

The routing decision:

| Signal | Meaning | Action |
|---|---|---|
| `SW Power Cap` (0x4) only | Intentional watts-for-FLOPs trade | None, or raise the cap if the rack allows |
| `SW Thermal Slowdown` (0x20) | Cooling shortfall — heat is not leaving fast enough | Facilities ticket; walk the §9 thermal path from the facility loop upward |
| `HW Thermal Slowdown` (0x40) | Emergency — clocks cut by ≥2× | Page. Cordon the node before it shuts down mid-job |
| `HW Power Brake` (0x80) | The PSU/system asserted the brake pin | Power design / PSU investigation |
| Xid / SXid in `dmesg` | A component actually faulted | **Cordon the node and file a hardware repair ticket.** Not a tuning problem |

Both can share a root cause — thermal stress in a dense rack can produce a throttle *and*, if it goes far enough, an Xid — but they are different rungs on the same ladder, and only one of them means "file a repair ticket."

### 13. Operationalising it

Production shops do not wait for a job to report low throughput. They run **passive health checks** continuously: a periodic sweep combining `dcgmi` diagnostics with `dmesg` scanning, looking specifically for thermal violations, sync-boost violations, hardware slowdowns and excessive temperatures. Modal's fleet-operations write-up describes exactly this pattern at 20,000-GPU scale and cites **>88 °C** as a practical operational red-flag threshold before scheduling new work onto a node.

The building blocks:

```
# Level 1 — the fast sweep, safe to run between jobs (seconds)
$ dcgmi health --check -g 0
Health Monitor Report
+------------------+---------------------------------------------------------+
| Overall Health: Warning                                                     |
+------------------+---------------------------------------------------------+
| GPU ID: 3        | Warning                                                  |
|                  | Thermal      : Detected thermal violations totalling     |
|                  |                412 seconds                               |
+------------------+---------------------------------------------------------+

# Level 2 — diagnostics; -r 1 is quick, -r 3 is a long stress run (minutes)
$ dcgmi diag -r 1

# Level 3 — the fault layer, always paired with the above
$ dmesg -T | grep -iE 'xid|sxid'

# Continuous telemetry — the fields from §5, scraped by dcgm-exporter
$ dcgmi dmon -e 100,112,150,153,155,156 -c 5
#Entity   SMCLK   CLEVT     GPUTMP  TMPMGN   POWER    TOTENRG
    GPU 0  1590  0x00000004     71      16   699.8  1.284e+09
    GPU 1  1980  0x00000000     64      23   641.2  1.191e+09
```

That `dmon` output is the whole lesson in one table: GPU 0 is power-capped (`0x4`) at 1590 MHz with 16 °C of thermal margin; GPU 1 is unconstrained at boost. If they are in the same job, GPU 1 is waiting for GPU 0 at every collective — which, in a sync-boost group, would instead show as `0x10` on GPU 1.

Design your alerts on three things, in this order:

1. **`DCGM_FI_DEV_GPU_TEMP_MARGIN_CELSIUS` (153) below a threshold** — SKU-independent, so one rule works across a mixed fleet.
2. **The event mask matching `0x20 | 0x40 | 0x80`** — thermal or power-brake bits, i.e. the *unchosen* constraints. Deliberately exclude `0x4`, or you will page on every correctly-capped GPU in the building.
3. **Any Xid/SXid entry** — a different severity and a different runbook.

And record the ratio you actually care about for cost: `clocks.sm / clocks.max.sm`, aggregated per node and per rack. **A rack whose mean clock ratio is 0.80 is delivering 80% of the FLOPs you are paying for, and no other single number says that.**

## Perspectives

**Developer.** Throttling is invisible from training code — a developer sees "throughput is lower than expected" with no signal distinguishing power/thermal from a data or communication bottleneck, because framework profilers report step time, not clock state. The one thing a developer *can* do is shape the workload: the activity factor α is theirs, and a kernel schedule that smooths power (the Google Cloud result in §11) is a code-level fix to a facility-level problem.

**Operator / facilities.** Power and cooling are shared, finite resources at the rack, row and facility level. Your job includes setting deliberate power caps to protect the circuit, and then checking whether *observed* throttling matches the *intended* cap. A mismatch — thermal throttling while the power budget still has headroom — means the cooling design is under-provisioned for the deployed density. That is a capacity-planning finding, not a per-job bug, and §9's series-resistance model tells you which stage to investigate.

**Hardware / silicon.** DVFS is the chip protecting itself. Every event reason is the silicon trading clock frequency for staying inside a physical constraint — power delivery capacity, or junction/HBM temperature. None of it is a fault. That reframes "why is my GPU throttled" from a fault-finding question into a capacity-planning question, and reframes an Xid error as the genuinely different case where something *did* break.

**Economics / facility design.** Two numbers matter and they are different. Per-GPU: a clock ratio of 0.80 is 20% of a $2–3/GPU-hr asset undelivered. Facility-wide: synchronized ML power draw is different enough from traditional datacenter load that it shortens the life of rectifiers, transformers and UPS batteries — a capital-equipment-lifecycle cost, not a throughput one. A platform engineer who can speak to both is arguing from the full picture an infra-economics interviewer wants to hear.

## Real-world use cases

- **Google Cloud — "Mitigating power and thermal fluctuations in ML infrastructure."** Documents cluster-level synchronized power swings of **tens of megawatts, near-instantaneously**, and a compiler-level workload-signature technique that cut power fluctuation by **~50%** and per-chip hotspot temperature swings from **~20 °C to ~10 °C**, at **under 1% performance overhead**. **What it shows:** the facility-level version of this lesson's die-level story, with a concrete, measured software mitigation — and proof that the interesting failure mode at scale is *timing*, not steady state. `https://cloud.google.com/blog/topics/systems/mitigating-power-and-thermal-fluctuations-in-ml-infrastructure`
- **Modal — "Keeping 20,000 GPUs healthy."** Documents the Xid/SXid error taxonomy in production, names **Xid 79 ("GPU has fallen off the bus")**, and describes the `dcgmi` + `dmesg` passive health-check pattern — including an **~88 °C** operational temperature threshold — used to catch thermal and hardware issues before they hit a running job. **What it shows:** the adjacent hardware-fault diagnostic layer and how it is operationalised continuously at fleet scale, which is the model answer for the capstone's "what exporter would you build" prompt. `https://modal.com/blog/gpu-health`
- **Meta Engineering — "How Meta keeps its AI hardware reliable."** Describes FBAR (Facebook Auto Remediation), Meta's automated-remediation system, and names **thermal runaway** explicitly as a transient-error class it detects and mitigates at fleet scale alongside GPU and HBM3 failures. **What it shows:** the reliability-engineering framing this lesson's throttle-versus-fault distinction feeds into — at Meta's scale the distinction is encoded in an automated remediation policy, not a human judgement call. `https://engineering.fb.com/2025/07/22/data-infrastructure/how-meta-keeps-its-ai-hardware-reliable/`
- **NVIDIA — DGX SuperPOD H100 data-center design guide, electrical specifications.** Publishes the rack-level design this lesson's §8 arithmetic reproduces: **415 V three-phase Wye, 32 A, ~21.8 kW per circuit**, three rPDUs on independent circuits in an N+1 scheme where no circuit carries more than 50% of the load, at **4 DGX H100 per rack (>40 kW)**. **What it shows:** that the whiteboard arithmetic and the vendor reference design agree — and exactly how little headroom a 40 kW rack has. `https://docs.nvidia.com/dgx-superpod/design-guides/dgx-superpod-data-center-design-h100/latest/electrical.html`

## Worked example

Load a GPU, read the state, cause a throttle on purpose, watch the clock move, then reason one level up to the rack. Transcripts are **representative** — absolute values depend on your SKU, driver, cooling and kernel — but every field name, flag and bit value is real.

### Step 1 — establish the envelope before loading anything

```
$ nvidia-smi --query-gpu=index,name,clocks.max.sm,power.default_limit,power.min_limit,power.max_limit \
             --format=csv
index, name, clocks.max.sm [MHz], default_power_limit [W], min_power_limit [W], max_power_limit [W]
0, NVIDIA H100 80GB HBM3, 1980 MHz, 700.00 W, 200.00 W, 700.00 W

$ nvidia-smi -q -i 0 -d TEMPERATURE | grep -E 'Shutdown|Slowdown|Max Operating|Current'
        GPU Current Temp                  : 34 C
        GPU Shutdown T.Limit Temp         : -5 C
        GPU Slowdown T.Limit Temp         : -2 C
        GPU Max Operating T.Limit Temp    : 0 C
        Memory Current Temp               : 38 C
        Memory Max Operating T.Limit Temp : 0 C
```

Now you know the envelope you are working inside: boost 1980 MHz, a 700 W default limit settable down to 200 W, and thermal thresholds expressed in T.Limit (margin) form. **Record the format** — an exporter that assumes absolute temperatures will misread this node.

### Step 2 — baseline under sustained load

Use a real training step, or a synthetic load such as `dcgmproftester` or `gpu-burn`. **Run it for at least 10 minutes** — §6 explains why 60 seconds is not enough.

```
$ dcgmproftester12 --no-dcgm-validation -t 1004 -d 600 &     # sustained tensor load

$ nvidia-smi --query-gpu=clocks.sm,clocks.max.sm,power.draw,enforced.power.limit,\
temperature.gpu,temperature.memory,clocks_event_reasons.active \
--format=csv,noheader,nounits
# t = 30 s
1980, 1980, 641.2, 700.00, 52, 58, 0x0000000000000000
# t = 120 s
1785, 1980, 699.6, 700.00, 68, 74, 0x0000000000000004
# t = 600 s
1755, 1980, 699.8, 700.00, 74, 81, 0x0000000000000004
```

Read the progression against §6's timeline:

- **t = 30 s** — mask `0x0`, clock at boost 1980, power 641 W with 59 W of headroom. Phase 1. This is the only sample that shows full rated throughput, and it is exactly the sample a short benchmark would report.
- **t = 120 s** — mask `0x4` (`SW Power Cap`), power pinned at 699.6/700.00, clock down to 1785 MHz. Phase 2. **The GPU is power-limited, not heat-limited** — both thermal bits are clear and there is still margin.
- **t = 600 s** — steady state at 1755 MHz. Die 74 °C, memory 81 °C. Still no thermal bit; the cooling on this box is adequate. If `0x20` had appeared here you would have a facilities finding.

Convert to the number that matters:

```
  clock ratio      = 1755 / 1980                        = 0.886
  delivered BF16   = 989.4 TFLOPS × 0.886               = 876.6 TFLOPS
  deficit          = 112.8 TFLOPS                       = 11.4% of rated

  and note: nvidia-smi's utilization.gpu read 100% at all three samples.
```

### Step 3 — cause a throttle on purpose and watch DVFS respond

```
$ sudo nvidia-smi -pm 1
Enabled persistence mode for GPU 00000000:9D:00.0.

$ sudo nvidia-smi -i 0 -pl 400
Power limit for GPU 00000000:9D:00.0 was set to 400.00 W from 700.00 W.

$ nvidia-smi --query-gpu=clocks.sm,power.draw,enforced.power.limit,temperature.gpu,\
clocks_event_reasons.active --format=csv,noheader,nounits
1245, 399.4, 400.00, 61, 0x0000000000000004
```

Trace it:

- Enforced limit is now 400 W and draw pins at 399.4 W — DVFS converged on an operating point that fits.
- SM clock fell **1755 → 1245 MHz**, a 29% cut, for a **43% cut in power** (699.8 → 399.4 W). That asymmetry is the `P ∝ V²f` relationship from §1 made visible: you gave up 29% of throughput to save 43% of watts.
- Die temperature dropped 74 → 61 °C, because you removed 300 W of heat from the same cooling path. Free proof of §9's series-resistance model.
- Reason bit is still `0x4` — the *same* bit, now representing a limit you set rather than the SKU default. **The bitmask tells you which constraint binds, not whether you chose it.** That judgement is yours.

Delivered throughput at the cap:

```
  1245 / 1980 = 0.629  →  989.4 × 0.629 = 622.3 TFLOPS BF16
  Perf-per-watt: 622.3 TFLOPS / 400 W  = 1.556 TFLOPS/W
                 876.6 TFLOPS / 700 W  = 1.252 TFLOPS/W
```

**The capped GPU is 24% more efficient per watt.** That is the argument for power capping when your constraint is total facility power rather than per-GPU throughput — you get more aggregate FLOPs out of a fixed kilowatt budget by running more GPUs slower than fewer GPUs at boost. It is also the argument *against* capping when your constraint is job wall-clock time. Which one applies is a business question, and you should be able to state both sides with these numbers.

Restore when done:

```
$ sudo nvidia-smi -i 0 -pl 700
Power limit for GPU 00000000:9D:00.0 was set to 700.00 W from 400.00 W.
```

### Step 4 — rule out the fault layer

```
$ dmesg -T | grep -iE 'xid|sxid'
$ echo $?
1
```

No output, exit status 1 — no Xid or SXid entries during the exercise. **Record this explicitly**, because the negative result is what licenses the conclusion "this was a throttle, not a fault." A real Xid 79 appearing during a throttle investigation changes the fix from "retune the cap" to "cordon the node and file a repair ticket."

### Step 5 — reason one level up, to the rack

Take the §7 and §8 arithmetic and apply it to what you just measured:

```
  Measured, one GPU at the 700 W default limit, thermal steady state:
    board power 699.8 W, module power (nvidia-smi -q -d POWER) ~731 W

  Node projection:
    8 × 731 W (module)                                 = 5,848 W
    + non-GPU load from the §7 budget                  ≈ 4,600 W
    ────────────────────────────────────────────────────────────
    ≈ 10.4 kW   — consistent with the 10.2 kW datasheet maximum ✓

  Rack, 4 nodes, three 21.8 kW circuits, N+1:
    normal:  40.8 / 3 = 13.6 kW per circuit  =  62% of 21.8 kW  ✓
    N-1:     40.8 / 2 = 20.4 kW per circuit  =  94% of 21.8 kW  ✓ (tight)

  Now the scenario that gets people fired:
    someone adds a fifth node "because there's rack space"
    normal:  51.0 / 3 = 17.0 kW per circuit  =  78%   ✓ looks fine
    N-1:     51.0 / 2 = 25.5 kW per circuit  = 117%   ✗ THE SURVIVING
                                                        CIRCUIT TRIPS
    → a single circuit failure now takes out the ENTIRE rack instead of
      being ridden through. The failure only manifests during the event
      you installed redundancy for.

  Cooling check for the 4-node rack, air-cooled at ΔT 15 K:
    V̇ = 40,800 / (1,206 × 15) = 2.26 m³/s = 4,780 CFM through the cabinet
    If the containment delivers only 3,500 CFM, the achievable ΔT rises to
    40,800 / (1,206 × 1.65) = 20.5 K — inlet 27 °C becomes exhaust 47.5 °C,
    the recirculated inlet climbs, and the top-of-rack nodes start reporting
    0x20 SW Thermal Slowdown first. Nothing in the GPU telemetry says "airflow";
    it says "thermal margin is shrinking."
```

**The rack-scale extension on the power side.** Now reason past steady state: if this box's neighbours are all training the same job and hit the same step boundary in lockstep, they all ramp together. Even if *every individual GPU* respects its `enforced.power.limit` at every sample — none ever shows `SW Power Cap` on a per-sample basis — the rack PDU sees a near-simultaneous multi-kilowatt swing, repeated every step, for the whole run. That is Google Cloud's phenomenon in miniature: no single-GPU telemetry catches it, because it is a property of *when* the ramps happen in aggregate. The single-GPU worked example above is necessary but not sufficient evidence at scale.

**Conclusion for the deliverable:** *At the 700 W default limit this GPU held 1755 MHz in thermal steady state under sustained tensor load — an 11.4% clock deficit against the 1980 MHz boost, entirely attributable to `SW Power Cap` (0x4), with 13 °C of thermal margin remaining and no thermal bit ever set. Capping to 400 W drove the clock to 1245 MHz (−29%) for a 43% power saving, improving perf-per-watt from 1.252 to 1.556 TFLOPS/W. No Xid/SXid entries appeared. Node power projects to ~10.4 kW, consistent with the 10.2 kW published maximum; at 4 nodes per rack on three 21.8 kW N+1 circuits the rack runs at 94% of a single circuit's capacity under N-1, leaving no room for a fifth node.*

## Practice

Feeds the [Topology Teardown](../practice/topology-teardown/README.md) deliverable. **Do the power-cap experiments only on a dedicated instance** — they are safe on a rented box you own end to end, and antisocial on shared hardware.

1. **Capture the envelope.** Before loading anything, record per GPU: `clocks.max.sm`, `power.default_limit`, `power.min_limit`, `power.max_limit`, and the full `nvidia-smi -q -d TEMPERATURE` block. Note whether your driver reports absolute temperatures or `T.Limit` margins, and say what that implies for an alert rule.
2. **Baseline under sustained load, long enough to reach thermal steady state.** Run a real training step or `dcgmproftester`/`gpu-burn` for **≥10 minutes**, sampling every 15 s. Record `clocks.sm`, `clocks.max.sm`, `power.draw`, `enforced.power.limit`, `temperature.gpu`, `temperature.memory`, and `clocks_event_reasons.active` at t = 30 s, t = 2 min and t = 10 min. Identify which of §6's phases each sample is in, and explicitly note whether a 60-second benchmark would have reported a different answer.
3. **Compute the FLOPs deficit.** From `clocks.sm / clocks.max.sm` and your GPU's rated dense throughput, compute delivered TFLOPS and the deficit in both TFLOPS and percent. State which constraint the bitmask says is binding, and — separately — whether that constraint was *chosen*.
4. **Cause a throttle deliberately.** Enable persistence mode, then `sudo nvidia-smi -pl <below the observed draw>` (e.g. 400 on a 700 W part). Re-read the whole block. Confirm `SW Power Cap : Active` / bit `0x4`, record the new clock, and compute **both** the throughput loss and the perf-per-watt change. If you have a throughput figure (samples/sec, tokens/sec), record it before and after and check that it tracks the clock ratio. Restore the original limit.
5. **Decompose any `HW Slowdown`.** If bit `0x8` ever appears, decompose it against `0x40` (thermal) and `0x80` (power brake) and say which. If it never appears, say so — a clean negative is a result.
6. **Check the fault layer.** Run `dmesg -T | grep -iE 'xid|sxid'` before and after. Record the result either way, and write one sentence on how a real Xid 79 would change the remediation from "retune the cap" to "cordon and file a repair ticket."
7. **Do the node and rack arithmetic for your hardware.** Build a component power budget for your node like §7's, anchored on your system's published maximum. Then compute your rack's per-circuit capacity from `√3 × V × I × PF`, the node count that fits under N-1 with no circuit above 50%, and the airflow (`V̇ = Q / (ρ·c_p·ΔT)`) or coolant flow (`ΔT = Q × 60 / (c_p × flow)`) the resulting heat load requires. State the assumptions you used.
8. **Design the alert.** Specify the exporter you would build: which DCGM field IDs, which bits of the event mask you page on and which you deliberately ignore, and why `DCGM_FI_DEV_GPU_TEMP_MARGIN_CELSIUS` (153) is a better alert basis than an absolute temperature across a mixed fleet. This is a direct rehearsal for lesson 08's Part B.

**Acceptance:** a note in the Topology Teardown showing at least one throttle reason observed with its bit and name; the clock delta it caused in MHz and as a ratio; the resulting TFLOPS deficit; the perf-per-watt figures before and after the cap; one sentence classifying the constraint as intentional (power cap) or symptomatic (thermal); confirmation that `dmesg` showed no Xid/SXid faults during the run (or documentation of one if it did); your node and rack power/cooling arithmetic with assumptions stated; and the alert design. Paste the relevant `nvidia-smi -q` and `dcgmi dmon` output.

## Common pitfalls

1. **Treating any event reason as a bug.** Every bit is DVFS working correctly under a real constraint. The question is never "is this broken" but "which constraint is binding, and did I choose it?" `0x4` on a correctly-provisioned rack is normal and expected.
2. **Trusting `GPU-Util: 100%` as proof of full throughput.** Utilization means a kernel was resident during the sample window. It says nothing about clock. Always pair it with `clocks.sm` versus `clocks.max.sm` — that ratio, not utilization, is what multiplies your rated TFLOPS.
3. **Reading `HW Slowdown` (0x8) as a cause.** It is a roll-up bit with three documented triggers. Decompose against `0x40` (thermal) and `0x80` (power brake) before routing the ticket, or you will send a power-supply problem to the cooling team.
4. **Grepping for "Clocks Throttle Reasons" after a driver upgrade.** NVIDIA renamed the group to "Clocks Event Reasons" (`nvmlClocksEventReason*`), with the old names retained as deprecated aliases. Log and CSV consumers matching the old string silently stop matching. Match both, or use the numeric field.
5. **Misreading `T.Limit` fields as absolute temperatures.** Newer drivers report *margin to threshold*, which falls as the GPU heats up. An alert on "temperature > 85" against a T.Limit field will never fire; an alert on "T.Limit < 5" is the correct form.
6. **Ignoring HBM temperature.** `SW Thermal Slowdown` defends *both* GPU Max Operating and Memory Max Operating temperature. On HBM3 parts the memory stack often binds first, so a die at a comfortable 74 °C can still be throttling because memory is at its limit. Chart `DCGM_FI_DEV_MEMORY_TEMP_CELSIUS` (140) alongside 150.
7. **Characterising thermals with a short benchmark.** The power constraint binds in seconds; the thermal constraint binds in minutes, because it depends on the heat capacity of the whole path from die to facility loop. A 60-second run reports phase-2 numbers for a phase-3 reality. Run ≥10 minutes and watch `clocks.sm` converge.
8. **Setting a power limit without persistence mode.** Without it the driver unloads when the last client exits and every setting reverts — so a cap applied to protect a circuit silently disappears between jobs, while your capacity plan still assumes it.
9. **Budgeting a rack as `N × 8 × GPU_TDP`.** The GPUs are ~55% of a DGX H100's 10.2 kW. The other 45% — CPUs, DRAM, NVMe, NICs, NVSwitch, fans, VRM and PSU conversion losses — is real, mostly unitemised, and is what makes a rack trip.
10. **Sizing a rack to its normal-operation load rather than its N-1 load.** Three circuits at 21.8 kW look like 65 kW of capacity; the number that matters is 43.6 kW, and it only matters on the day you find out.
11. **Conflating throttling with an Xid/SXid fault.** Throttling is a policy response; an Xid is a fault event. They can share a root cause and they route to completely different remediations.
12. **Assuming per-GPU telemetry is sufficient at fleet scale.** The interesting failure at scale is *synchronized*, cluster-level power and thermal swings that no single GPU's event mask captures. You need rack- or facility-level power telemetry, and the instantaneous power field (157) rather than the averaged one (155), to see it at all.

## Self-check

**(a) Which event reason means "you are paying for cooling you do not have" versus "you capped power on purpose," and what are their hex values?**

**Answer:** `SW Thermal Slowdown` (`nvmlClocksEventReasonSwThermalSlowdown`, **`0x20`**) means the clock is limited because the GPU cannot shed heat fast enough — it is reducing clocks specifically so that GPU temperature stays under GPU Max Operating Temperature and memory temperature stays under Memory Max Operating Temperature. Nobody chose it; you are paying for cooling capacity you do not have. Its emergency tier is `HW Thermal Slowdown` (**`0x40`**), which cuts core clocks by a factor of 2 or more. `SW Power Cap` (**`0x4`**) means the clock is limited to keep board power under the currently enforced limit — an intentional watts-for-FLOPs trade that you or the facility made. The facility-driven `HW Power Brake Slowdown` (**`0x80`**) is an external assertion from the system power supply, also ≥2× clock cut. Thermal = symptom, page facilities; power cap = decision, verify it was yours. Note `HW Slowdown` (**`0x8`**) is a roll-up bit, not a cause — decompose it with `0x40` and `0x80`.

**(b) Utilization reads 100% but the SM clock is 1590 against a 1980 max. Is the GPU busy, and what is the cost?**

**Answer:** It is "busy" only in the sense that a kernel was resident during each sample window — that is all `utilization.gpu` measures. It is **not** delivering full throughput. Tensor throughput is linear in clock, so at 1590/1980 = 0.803 the GPU delivers `989.4 × 0.803 = 794.5 TFLOPS` dense BF16 against a rated 989.4 — a deficit of **194.9 TFLOPS, or 19.7%**. (1590 MHz is not an arbitrary number: it is the H100 SXM5's **base clock**, so this GPU has been pushed all the way back from boost to base.) You are paying the full hourly rate for ~80% of the silicon's rated output and the primary utilization dashboard reads unchanged. Only `clocks.sm` versus `clocks.max.sm` plus the event mask expose it.

**(c) Why does a fully populated rack deliver less than `N × single-GPU spec`? Name two mechanisms and how you would tell them apart.**

**Answer:** (1) **Shared power budget.** Circuit capacity is fixed — e.g. 21.8 kW per 415 V/32 A three-phase circuit — and the N-1 survivability rule consumes a large fraction of it, so operators cap per node or per GPU to stay under the breaker. Every GPU then runs with `SW Power Cap` (`0x4`) active, below boost, permanently. (2) **Shared cooling and thermal coupling.** `T_junction = T_facility_water + Σ ΔT` across TIM, cold plate, coolant loop and heat exchanger. A full rack raises the common terms for everyone: more aggregate heat means a warmer return, a warmer supply for the next node, and every die running hotter at the same power. `SW Thermal Slowdown` (`0x20`) appears first on the worst-positioned GPUs — middle of a chassis, top of a rack. Both drop clocks while per-GPU utilization still reads 100%. **The bitmask tells them apart:** `0x4` alone with thermal margin remaining is mechanism (1); `0x20` set is mechanism (2). And mechanism (2) with power headroom remaining is specifically a cooling under-provisioning finding, not a power one.

**(d) A 10.2 kW node, four to a rack, three 21.8 kW circuits N+1. Does it fit, and what happens if you add a fifth node?**

**Answer:** Four nodes are 40.8 kW, and the binding test is the N-1 case, not normal operation: losing any one circuit must leave the other two able to carry the whole load. `2 × 21.8 = 43.6 kW ≥ 40.8 kW` ✓ — it fits, at 94% loaded under N-1, with each circuit carrying 13.6 kW (62%) normally. Add a fifth node and you have 51.0 kW: normal operation is 17.0 kW per circuit (78%, which is exactly why nobody notices), but under N-1 each surviving circuit must carry 25.5 kW against a 21.8 kW capacity — **117%, and the survivor trips.** A single circuit failure now takes out the entire rack instead of being ridden through. The failure manifests only during the precise event you installed redundancy for, which is what makes it so dangerous.

**(e) Why is water used for 120 kW racks and air for 40 kW ones? Derive it.**

**Answer:** From `Q = V̇ · ρ · c_p · ΔT`, the fluid property that matters is the **volumetric heat capacity** `ρ·c_p`. For air at ~20 °C that is `1.20 × 1005 = 1,206 J/(m³·K)`; for water it is `1000 × 4180 = 4,180,000 J/(m³·K)` — a ratio of about **3,470×**. Concretely: removing 120 kW with air at a 15 K rise needs `120,000 / (1,206 × 15) = 6.63 m³/s = 14,050 CFM` through one 600 mm cabinet, which is a wind tunnel, not a cooling design. Removing the same 120 kW with water at ~130 L/min gives `ΔT = 120 × 60 / (4.18 × 130) = 13.2 K` — a 13 K rise through roughly a one-inch pipe. A 40 kW rack needs `40,800 / (1,206 × 15) = 2.26 m³/s = 4,780 CFM`, which conventional hot-aisle containment can supply, so air is still viable there. The crossover is not a preference; it is where the airflow required stops being physically deliverable into a rack-sized volume.

**(f) What is the difference between a GPU throttling and a GPU throwing Xid 79, and why does it change the remediation?**

**Answer:** A throttle reason is DVFS working as designed — the silicon voluntarily trading clock for staying inside a power or thermal constraint. The GPU is healthy; the fix is a power-cap or cooling conversation, and you walk the thermal path from the facility loop upward. **Xid 79, "GPU has fallen off the bus,"** is a hardware/driver *fault* event surfaced by the NVIDIA driver in `dmesg`: the driver can no longer communicate with the device over PCIe. Thermal or power stress in a dense rack can be the trigger, but it is not the same signal. The remediations diverge completely: retune the cap or improve cooling for a throttle reason; **cordon the node and file a hardware repair ticket** for an Xid. `SXid` is the NVSwitch-specific equivalent on multi-GPU NVSwitch systems, where the fabric rather than a GPU is the faulting component. Treating one as the other wastes days chasing the wrong fix — which is why every throttle investigation must include `dmesg -T | grep -iE 'xid|sxid'`, and why a clean negative result is worth recording.

**(g) Why can synchronized ML training stress datacenter electrical infrastructure even when every accelerator stays under its power cap?**

**Answer:** Synchronous data- and tensor-parallel training makes thousands of accelerators hit the same step boundaries in near-lockstep, so their activity factor α — and therefore their power draw — rises and falls together rather than at independent, staggered times the way traditional uncorrelated datacenter workloads do. The result is a near-instantaneous, tens-of-megawatt aggregate swing at the cluster level, repeated every step for the whole run. Rectifiers, transformers and UPS batteries are specified for a load *magnitude*, not for that *rate of change*, so the damage is to equipment lifetime rather than to any instantaneous limit — and no single chip ever exceeds its `enforced.power.limit`, so no per-GPU telemetry shows anything wrong. Google Cloud's mitigation reshapes *when* compute blocks run within a step to smooth the aggregate curve, achieving ~50% less power fluctuation and ~20 °C → ~10 °C smaller hotspot temperature swings at under 1% performance overhead. The lesson for monitoring: use rack- or facility-level power telemetry, and the *instantaneous* DCGM power field (157) rather than the averaged one (155), because averaging removes exactly the transient you are hunting.

## Connections & what's next

Power and thermal throttling is the last of the host-side "GPU busy, throughput low" causes this module builds — after NUMA placement (lessons 01–02), PCIe link health (lesson 03), the reference 8-GPU layout (lesson 04), Kubernetes topology-aware scheduling (lesson 05), and storage placement (lesson 06). It also carries a second, distinct diagnostic layer, Xid/SXid, that §12 deliberately separates from throttling proper — remember both when you get to lesson 08's diagnostic tree, where a hardware fault is a genuinely different hypothesis with a genuinely different fix.

It also closes a loop back to lesson 06. `nvme smart-log`'s `warning_temp_time`, `critical_comp_time` and thermal-management transition counters are the storage-side equivalent of `SW Thermal Slowdown`: a device trading throughput for temperature, measured in wall-clock minutes, inside a chassis whose thermal budget the GPUs already dominate. And the §7 node budget is where lesson 06's eight NVMe drives at ~20 W each stopped being a rounding error.

Next is the **capstone**: reconcile all four topology tools into one diagram on a real node, then measure what a misalignment costs. Lesson 08's symptom list — "GPU at 100%, throughput half spec, no error" — is the full checklist this module has been building toward. NUMA placement, PCIe link state, GPU↔NIC path, NVMe placement, power/thermal state, and (as a distinct hypothesis) a hardware fault surfaced via Xid/SXid all belong on it, and this lesson supplied the last two entries plus the exporter design you will defend there.

## References & further reading

**Primary sources**

- **NVML API Reference — `nvmlClocksEventReasons` group** — `https://docs.nvidia.com/deploy/nvml-api/group__nvmlClocksEventReasons.html` — the authoritative bit definitions and hex values used in §2: `GpuIdle` 0x1, `ApplicationsClocksSetting` 0x2, `SwPowerCap` 0x4, `HwSlowdown` 0x8, `SyncBoost` 0x10, `SwThermalSlowdown` 0x20, `HwThermalSlowdown` 0x40, `HwPowerBrakeSlowdown` 0x80, `DisplayClockSetting` 0x100 — plus the documented multi-cause nature of `HwSlowdown` and the deprecation of the `nvmlClocksThrottleReason*` names in favour of `nvmlClocksEventReason*`. **Correction applied from this source:** earlier versions of this lesson treated `HW Slowdown` as a distinct cause; NVML documents it as a roll-up with three possible triggers (temperature, external power-brake assertion, fast-trigger power protection) that must be decomposed via bits 6 and 7.
- **NVML API Reference — `nvmlTemperatureThresholds_enum`** — `https://docs.nvidia.com/deploy/nvml-api/` — the four operational thresholds (`SHUTDOWN` 0, `SLOWDOWN` 1, `MEM_MAX` 2, `GPU_MAX` 3) and the fact that `MEM_MAX` triggers a *software* slowdown just as `GPU_MAX` does.
- **`nvidia-smi` manual / `-q` device query reference** — `https://docs.nvidia.com/deploy/nvidia-smi/index.html` — the `-d` query groups (`POWER`, `TEMPERATURE`, `CLOCK`, `PERFORMANCE`, …), the `clocks_event_reasons.*` / `clocks_throttle_reasons.*` CSV fields, `-pl` for the power limit, `-pm` for persistence mode, and the `T.Limit` output format.
- **NVIDIA H100 PCIe GPU Product Brief** — `https://www.nvidia.com/content/dam/en-zz/Solutions/gtcs22/data-center/h100/PB-11133-001_v01.pdf` — the concrete H100-generation thermal reference points used in §3: 87 °C GPU max operating, 89 °C hardware slowdown, 92 °C shutdown, 95 °C HBM thermal qualification. **Correction applied:** an earlier version of this lesson presented 1590 MHz as an arbitrary throttled clock; it is the H100 SXM5 **base clock**, so a GPU pinned there has been pushed the full distance from boost to base.
- **NVIDIA DGX H100 datasheet and system user guide** — `https://docs.nvidia.com/dgx/dgxh100-user-guide/` — 10.2 kW maximum system power, six 3,300 W Titanium PSUs in 4+2 redundancy at 200–240 V / 16 A, 5–30 °C operating inlet range, and the storage/network configuration used in the §7 budget.
- **NVIDIA DGX SuperPOD: Data Center Design Guide (H100), Electrical Specifications** — `https://docs.nvidia.com/dgx-superpod/design-guides/dgx-superpod-data-center-design-h100/latest/electrical.html` — 415 V three-phase Wye at 32 A giving ~21.8 kW per circuit, three rPDUs on independent circuits in an N+1 scheme with no circuit carrying more than 50% of the load, at 4 DGX H100 (>40 kW) per rack. The reference the §8 arithmetic is checked against.
- **NVIDIA DCGM — `dcgm_fields.h`** — `https://github.com/NVIDIA/DCGM/blob/master/dcgmlib/dcgm_fields.h` — the field IDs in §5, including `DCGM_FI_DEV_CLOCKS_EVENT_REASONS` (112, with `DCGM_FI_DEV_CLOCK_THROTTLE_REASONS` as a deprecated alias), `DCGM_FI_DEV_GPU_TEMP_MARGIN_CELSIUS` (153), `DCGM_FI_DEV_TOTAL_ENERGY_CONSUMPTION` (156, in mJ), and the 160–164 power-limit hierarchy.

**Real-world engineering**

- **Google Cloud — "Mitigating power and thermal fluctuations in ML infrastructure"** — `https://cloud.google.com/blog/topics/systems/mitigating-power-and-thermal-fluctuations-in-ml-infrastructure` — **what it shows:** synchronized fleet-wide power ramping measured in tens of megawatts, and a compiler-level workload-shaping mitigation delivering ~50% less fluctuation and ~20 °C → ~10 °C smaller hotspot swings at under 1% overhead.
- **Modal — "Keeping 20,000 GPUs healthy"** — `https://modal.com/blog/gpu-health` — **what it shows:** the Xid/SXid taxonomy in production, Xid 79 by name, the `dcgmi` + `dmesg` passive health-check pattern, and the ~88 °C operational red-flag threshold.
- **Meta Engineering — "How Meta keeps its AI hardware reliable"** — `https://engineering.fb.com/2025/07/22/data-infrastructure/how-meta-keeps-its-ai-hardware-reliable/` — **what it shows:** FBAR automated remediation with thermal runaway as an explicitly named fleet-scale failure class, i.e. the throttle-versus-fault distinction encoded as policy rather than judgement.

**Deeper dives**

- **Brendan Gregg, *Systems Performance*, 2nd ed.** — `https://www.brendangregg.com/systems-performance-2nd-edition-book.html` — the CPU DVFS and power-management chapters for the general model of frequency scaling under a physical constraint; read the GPU as a fast-DVFS instance of the same mechanism.
