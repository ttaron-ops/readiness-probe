---
lesson: "03.7"
title: "Capstone — cost per unit of useful work"
module: "03"
concept: "Cost-per-useful-work synthesis"
status: not-started
est_time: "10h"
prev: "06-generational-and-software-stack.md"
next: null
artifacts: []
sources: 16
---

# 03.7 · Capstone — cost per unit of useful work

> **Concept.** Combine achieved-vs-spec TFLOPS, the FP16→FP8 delta, the "utilisation lies" demo, and **tier-aware live $/hr** into a defensible cost-per-useful-work analysis and a SKU recommendation — and learn why the tier you price against can flip the recommendation.
>
> Module: [🔌 03 — GPU hardware fundamentals](../README.md) · Deliverable: [GPU Efficiency & Cost Report](../practice/gpu-efficiency-report/README.md)

## Where this fits

Six lessons built six separate facts. L1 taught you that `GPU-Util` measures kernel *residency*, not useful work — the util-lie. L2 gave you the roofline: whether a workload is compute-bound or memory-bound, and how far it sits from its ceiling. L3–L4 gave you the memory-hierarchy arithmetic — what fits in HBM, and why decode's tokens/sec ceiling is a bandwidth number, not a compute number. L5 gave you the precision lever, the numeric formats behind it, and the `SMs × FLOP/SM/clock × clock` identity that says exactly where a 2× comes from. L6 gave you the generational SKU table (A100/H100/H200/B200), the decision tree that maps a bottleneck onto a column of it, and the operational discipline that keeps a fleet from silently underperforming its spec.

Each of those facts is trivia in isolation. This capstone welds them into the one artifact a GPU-heavy platform team actually transacts on: **cost per unit of useful work** — `$/1M tokens` for serving, `$/useful-PFLOP` for compute-bound work — and a defended SKU recommendation. It also adds the one input none of the prior six lessons touched: **price**, and specifically the fact that price is not a single number per SKU. This lesson is where "I understand GPU hardware" becomes "I can tell you which box to buy, and defend the number in a room."

Unlike the previous six, this lesson is a **procedure**. Everything below is written so you can execute it end to end on one rented GPU over one weekend, in order, without opening a tutorial.

## Why this matters

Every one of this module's job signals — CoreWeave's "hardware validation across the fleet," NVIDIA's "arithmetic intensity vs peak compute using the roofline," the interview question "which SKU is cheaper per unit of useful work?" — collapses into this lesson. It is, verbatim, the last row of the module's own lesson table: **"the SKU recommendation, defended by numbers."**

The stake is concrete. A platform team renting GPU capacity is making a recurring six- or seven-figure decision, and "which GPU should we buy/rent" gets asked in nearly every infra review at a company running model training or inference at any scale. Most engineers answer it with a spec sheet or a sticker price — "H100 is $X/hr, H200 is $Y/hr, so..." — and stop there. That answer is wrong often enough to matter: a GPU that is 40% more expensive per hour can be cheaper per token, and a GPU that is cheaper per hour can be *more* expensive per token, depending on what the extra memory or bandwidth buys in throughput. The engineer who can only reason in $/hr loses that argument to the engineer who reasons in $/token — and in an interview, "why is the H200 sometimes the better inference buy despite costing more?" is precisely the checkpoint's own depth probe.

The cost of *not* knowing this shows up as real money: a team that buys on $/hr alone routinely overpays for a given workload, and a report built on stale or tier-blind pricing gets a wrong SKU call through the door because nobody checked which reliability tier the number came from. Both failure modes are avoidable, and both are exactly what this lesson trains you out of.

## What's new here (calibration)

The module's calibration note still holds: you are not becoming a CUDA kernel author, and NVLink/PCIe topology and power/thermal throttling stay in 02b's territory, referenced not re-taught. For this capstone specifically:

- **Not new:** the four technical inputs (achieved-vs-spec TFLOPS, the FP16→FP8 delta, the util-lie, the memory/bandwidth table) — you built all four in L1–L6. This lesson does not re-derive them; it *sequences the measurements* and shows the arithmetic that turns them into money.
- **Genuinely new: the pricing model itself.** Every prior lesson treated `$/hr` as a single placeholder number. This lesson replaces that fiction with the real market structure — GPU-cloud pricing is **tiered by reliability and support and stratified by contract term**, not a single figure per SKU, and the tier you compare against changes the answer.
- **Genuinely new: the end-to-end measurement procedure.** §3–§7 below are a runnable sequence — provision, verify, measure, price, conclude — with the exact commands, the expected output shape, and the arithmetic that consumes each measurement. Nothing in it requires an external tutorial.
- **Genuinely new: sensitivity analysis as a discipline.** Producing *one* cost-per-token number is the easy 80% of this lesson. Showing how that number moves when you change one input (here, pricing tier) — and stating which conclusions are robust to that change and which aren't — is the staff-level habit this capstone is built to install.
- **Genuinely new: turning a technical result into a document a non-engineer will act on.** The report is the deliverable; a number a finance stakeholder can't audit isn't done.

## Core concepts

### 1. What "useful work" is, and the two denominators

The whole method rests on a definition. **Useful work is the unit your organisation sells or consumes — not a unit of machine activity.** For an inference platform that is a *token*. For a training or HPC platform it is a *floating-point operation the model actually needed*. Everything the GPU does that is not one of those — spinning in a launch-bound kernel, waiting on HBM, re-reading weights it already read, running an all-reduce that a bigger GPU would not have needed — is overhead you are paying for.

That gives two cost expressions and one rule for choosing between them:

```
  SERVING (decode-dominated, memory-bound):

      $/1M tokens  =  ($/hr)  /  (tokens_per_sec × 3600 / 1e6)

      tokens_per_sec is AGGREGATE output throughput across all concurrent
      requests on the GPU (or the whole TP group), measured at the batch
      size that still meets your latency SLO. Not per-request. Not prefill.

  COMPUTE-BOUND (training steps, large GEMMs, prefill-heavy):

      $/useful-PFLOP  =  ($/hr)  /  (achieved_PFLOP_per_sec × 3600)

      achieved_PFLOP/s is measured, not spec. The ratio of the two is MFU.

  WHICH ONE?  Ask lesson 2's question: which roof is the workload under?
      compute-bound → $/useful-PFLOP is meaningful, MFU is the efficiency metric
      memory-bound  → $/1M-tokens is meaningful, and BANDWIDTH utilisation
                      (achieved GB/s ÷ spec GB/s) is the efficiency metric.
                      Compute-MFU in the single digits is CORRECT here, not a bug.
```

The second half of that last line is the most common misreading in GPU reports. A decode server at 4% compute-MFU is not broken; it is memory-bound, exactly as L3–L4 predict, and the Google Cloud B200 benchmark (96 GPUs, ~1M tok/s aggregate, 4.4% FLOPS utilisation and 1.5% tensor-active at decode batch=1) is the published production confirmation. **Report the metric that matches the roof.** Reporting compute-MFU for a decode workload and calling it "poor efficiency" is how a report loses its author's credibility in the first five minutes.

### 2. The shape of the method

```
  WHAT FEEDS WHAT — every prior lesson supplies exactly one input

  L1  util-lie demo ────────────▶ credibility check: the number you report is
      (GPU-Util vs SM/tensor)      "useful work", not "the GPU was busy"
                                           │
  L2  roofline placement ──────────────────┼──▶ chooses WHICH denominator
      (compute roof or memory roof)        │    ($/PFLOP vs $/1M-tok)
                                           │
  L2  achieved TFLOPS ─────┐               │
  L5  dense spec TFLOPS ───┴──▶ MFU ───────┤
                                           │
  L3  what fits (weights+KV) ──┐           │
  L4  decode ceiling + batch ──┴──▶ achieved tok/s ──┐
                                                     │
  L5  FP8 vs FP16 delta ─────────────────────────────┤
      (bytes halved, ~2× TC rate,                    │
       KV headroom → bigger batch)                   │
                                                     ├──▶  $/1M tokens
  L6  SKU choice (H100 / H200 / B200)                │     $/useful-PFLOP
      + stack sanity (versions, throttle) ───────────┤
                                                     │
  THIS LESSON: tier-aware, dated, term-matched $/hr ─┘
                                                     │
                                                     ▼
                                     SKU RECOMMENDATION + SENSITIVITY
                                     ("robust to tier?" "robust to term?")
```

### 3. The measurement plan — one rented GPU, one weekend

Everything in this capstone is measurable in **7–8 GPU-hours**. At a 2026 low-tier on-demand rate of roughly $2–3/GPU-hr that is about **$15–25 of compute**, which is what the module README's "$25–35" budget covers with headroom for a false start. Run it in this order — later steps depend on earlier ones, and the ordering minimises how long you hold an idle rented GPU.

```
  THE WEEKEND, IN ORDER          (elapsed GPU time, cumulative)

  ├─ 0:00  PROVISION + VERIFY            (~20 min)   §4
  │        rent → ssh → stack inventory → Xid/throttle baseline
  │        GATE: if the stack is wrong, everything below is garbage.
  │
  ├─ 0:20  A. COMPUTE CEILING            (~40 min)   §5.A
  │        GEMM sweep BF16 + FP8 → achieved TFLOPS → MFU vs DENSE spec
  │
  ├─ 1:00  B. MEMORY CEILING             (~20 min)   §5.B
  │        device-to-device bandwidth → achieved GB/s → % of HBM spec
  │        (these two give you the ROOFLINE AXES for your actual card)
  │
  ├─ 1:20  C. THE UTIL LIE               (~30 min)   §5.C
  │        1-block kernel: GPU-Util ≈100%, SM_ACTIVE ≈0.8%, TENSOR ≈0
  │        capture side-by-side with timestamps → the screenshot
  │
  ├─ 1:50  D. MODEL LOAD + FIT CHECK     (~40 min)   §5.D
  │        download weights, load at FP8, record actual HBM used vs predicted
  │
  ├─ 2:30  E. DECODE + BATCH CURVE       (~3 h)      §5.E
  │        single-stream tok/s vs bandwidth-derived ceiling;
  │        then sweep concurrency 1→256 until KV capacity or SLO caps it
  │
  ├─ 5:30  F. FP16 vs FP8 DELTA          (~1.5 h)    §5.F
  │        re-run E at the other precision; memory + bandwidth + tok/s deltas
  │
  ├─ 7:00  G. TEARDOWN                   (~15 min)
  │        scp the raw logs OFF the box, then DESTROY the instance
  │
  └─ 7:15  H. OFF-GPU: PRICE + ARITHMETIC + WRITE-UP   (~3 h, no GPU)  §6–§8
```

**The single most expensive mistake in this whole exercise is leaving the instance running while you write.** Steps A–F produce log files; §6 onward is arithmetic on those files. Destroy the box first.

### 4. Provision and verify — the gate

Rent one H100 or H200 (FP8 needs Hopper+, per L5 §15). Then, before measuring anything, establish that the machine is what you paid for and the stack is coherent — L6's discipline applied as a precondition:

```bash
# --- identity and spec -------------------------------------------------
nvidia-smi --query-gpu=index,name,driver_version,memory.total,compute_cap,\
clocks.max.sm,power.limit --format=csv
#  index, name,            driver_version, memory.total, compute_cap, clocks.max.sm, power.limit
#  0,     NVIDIA H100 80GB HBM3, 570.124.06, 81559 MiB,  9.0,         1980 MHz,      700.00 W

# --- the stack inventory (L6 §5) ---------------------------------------
python -c "import torch; print('torch', torch.__version__, '| cuda', torch.version.cuda,
           '| cudnn', torch.backends.cudnn.version(),
           '| nccl', '.'.join(map(str, torch.cuda.nccl.version())))"
nvcc --version | tail -2
cat /proc/driver/nvidia/version

# --- health baseline (L6 §9) -------------------------------------------
sudo dmesg -T | grep -i xid || echo "OK: no Xid errors at baseline"
nvidia-smi -q -d PERFORMANCE | grep -A 14 "Clocks Event Reasons"
nvidia-smi -q -d ECC | grep -A 4 "Aggregate"

# --- profiling plumbing (L1) -------------------------------------------
dcgmi discovery -l          # confirm DCGM sees the GPU
dcgmi profile -l            # confirm the profiling fields are available here
```

**Three gate conditions.** Fail any of them and fix it before spending money on measurements:

1. **`compute_cap` is 9.0 (Hopper) or higher.** 8.0 means you got an A100 and there is no FP8 path (L5 §15). Some marketplaces will silently give you a different SKU than the listing implied.
2. **No Xid errors, and `Clocks Event Reasons` shows only `GpuIdle` / `Not Active`.** A card already in `SW_Thermal_Slowdown` or `HW_Power_Brake` at idle is a card whose real roofline is below spec, and every percentage you compute against spec will be wrong in an interesting-but-unhelpful way (L6 §10).
3. **The driver meets the minimum for your toolkit's CUDA major family** (L6 §5: 12.x ≥ 525.60.13, 13.x ≥ 580.65.06). If `torch.cuda.is_available()` is `False` while `nvidia-smi` works, that is boundary (B) and you have a version problem, not a hardware problem.

**Record the power limit.** Rented H100s are sometimes provisioned at a reduced cap; a 500 W H100 will not reach the 1,830 MHz that the 989.5 TFLOPS spec assumes, and you want that on the record before you report a "poor" MFU.

### 5. The measurements

#### A — the compute ceiling: achieved vs spec TFLOPS

A large square GEMM is the cleanest available proxy for peak tensor throughput: high arithmetic intensity, no data-dependent branching, and a FLOP count you know exactly. For an `N×N` by `N×N` matmul, `FLOPs = 2N³` (one multiply and one add per MAC).

```python
# bench_gemm.py — writes results.csv; run as: python bench_gemm.py | tee raw/gemm.log
import torch, time, csv, sys

def timed(fn, warmup=10, iters=50):
    for _ in range(warmup): fn()
    torch.cuda.synchronize()
    t0 = time.perf_counter()
    for _ in range(iters): fn()
    torch.cuda.synchronize()
    return (time.perf_counter() - t0) / iters          # seconds per call

rows = []
for N in (2048, 4096, 8192, 16384):
    flop = 2 * N**3
    a = torch.randn(N, N, device="cuda", dtype=torch.bfloat16)
    b = torch.randn(N, N, device="cuda", dtype=torch.bfloat16)
    t = timed(lambda: torch.matmul(a, b))
    rows.append(("bf16", N, flop/t/1e12, t*1e3))

    a8 = a.to(torch.float8_e4m3fn)
    b8 = b.t().contiguous().t().to(torch.float8_e4m3fn)   # FP8 needs col-major B
    s  = torch.tensor(1.0, device="cuda")
    t = timed(lambda: torch._scaled_mm(a8, b8, scale_a=s, scale_b=s,
                                       out_dtype=torch.bfloat16))
    rows.append(("fp8", N, flop/t/1e12, t*1e3))
    del a, b, a8, b8; torch.cuda.empty_cache()

w = csv.writer(sys.stdout); w.writerow(["dtype","N","TFLOPs","ms"])
for r in rows: w.writerow([r[0], r[1], f"{r[2]:.1f}", f"{r[3]:.3f}"])
```

Representative output on an H100 SXM5 (**illustrative shape, not a captured transcript — your numbers replace these**):

```
dtype,N,TFLOPs,ms
bf16,2048,431.2,0.040
fp8,2048,502.7,0.034
bf16,4096,689.5,0.199
fp8,4096,982.1,0.140
bf16,8192,761.3,1.444
fp8,8192,1284.6,0.856
bf16,16384,772.0,11.395
fp8,16384,1310.4,6.713
```

Read it, and note the three things the shape tells you:

- **Small N is launch- and tail-bound.** At N=2048 you are nowhere near peak because the kernel is too short to fill the machine. Report your headline from the largest size that fits.
- **The FP8/BF16 ratio grows with N** (1.17× at 2048, 1.70× at 16384). The tensor-core multiple only materialises once the GEMM is big enough to be genuinely compute-bound. This is L5's roofline point measured directly: FP8 doubles the ridge point, so small matrices that were already borderline fall to the memory side.
- **Neither dtype reaches 100% of spec, and that is normal.** Published `cublasLt`-backed FP8 GEMM figures on H100 land in the ~1,200–1,300 TFLOPS region at this size.

Now the arithmetic — **always against the dense spec** (L5 §7; the 3,958 / 1,979 billboard figures are 2:4-sparsity numbers):

```
  H100 SXM5 dense spec:  BF16 = 989.5 TFLOP/s      FP8 = 1,979 TFLOP/s

  MFU(BF16) = 772.0 / 989.5 = 0.780  →  78.0% of dense peak
  MFU(FP8)  = 1310.4 / 1979  = 0.662  →  66.2% of dense peak

  measured FP8/BF16 multiple = 1310.4 / 772.0 = 1.70×
      (against a theoretical 2.00× — the gap is real and worth a sentence:
       at FP8 the same GEMM moves half the bytes for the same FLOPs, so it
       sits closer to the memory roof and the tensor cores stall more often)

  ⚠ If you had divided by the SPARSE spec instead:
       1310.4 / 3958 = 33.1%  — half the true efficiency, and any $/PFLOP
       downstream of it doubled. This single error is the most common way a
       GPU efficiency report becomes wrong.
```

**Also compute the effective-clock correction** if the card throttled during the sweep. Sample clocks alongside:

```bash
nvidia-smi --query-gpu=clocks.sm,temperature.gpu,power.draw,clocks_throttle_reasons.active \
           --format=csv -l 1 > raw/clocks_during_gemm.csv &
```

If the sustained SM clock was, say, 1,650 MHz rather than 1,830 MHz, the achievable BF16 peak on *that card* was `132 × 4096 × 1.65e9 = 892 TFLOPS`, and your 772 TFLOPS is 86.5% of achievable rather than 78.0% of spec. **Report both. They answer different questions:** "how good is my kernel" (vs achievable) and "am I getting the silicon I rented" (vs spec).

#### B — the memory ceiling: achieved bandwidth

```python
# bench_bw.py — device-to-device streaming bandwidth
import torch, time
N = 1 << 28                                   # 268M elements
a = torch.empty(N, device="cuda", dtype=torch.float16)   # 512 MiB
b = torch.empty_like(a)
for _ in range(5): b.copy_(a)
torch.cuda.synchronize(); t0 = time.perf_counter()
for _ in range(50): b.copy_(a)
torch.cuda.synchronize(); dt = (time.perf_counter() - t0) / 50
bytes_moved = 2 * a.numel() * a.element_size()           # one read + one write
print(f"{bytes_moved/dt/1e9:.0f} GB/s")
```

```
  representative H100 result:  2,610 GB/s
  bandwidth utilisation = 2610 / 3350 = 77.9% of HBM3 spec
```

70–85% of theoretical peak is the normal range for a streaming copy; DRAM refresh, row activation and the read/write turnaround make 100% unreachable. **You now have both roofline axes measured on the actual card**: `achieved_peak_FLOPS = 1,310 TFLOP/s (FP8)` and `achieved_BW = 2.61 TB/s`, giving a *measured* ridge point of `1310e12 / 2.61e12 = 502 FLOP/byte` against the spec ridge of 591. Plot your workloads against the measured roofline, not the datasheet one.

#### C — the util lie, captured

The demo is a kernel that occupies exactly one SM and does no useful math, run while both metrics are sampled:

```python
# util_lie.py — one block, one thread, spinning. Nothing useful happens.
import torch
src = r'''
extern "C" __global__ void spin(volatile long long *flag, long long iters) {
    for (long long i = 0; i < iters; ++i) { if (*flag) break; }
}
'''
# simplest portable equivalent without a custom kernel: a tiny serial loop
x = torch.zeros(1, device="cuda")
import time; t_end = time.time() + 120
while time.time() < t_end:
    for _ in range(2000):
        x = x + 1          # 1-element kernels: resident, useless, serialised
```

Capture both views for two minutes, in two other terminals:

```bash
nvidia-smi --query-gpu=timestamp,utilization.gpu,utilization.memory,clocks.sm \
           --format=csv -l 1 > raw/utillie_smi.csv

dcgmi dmon -e 1001,1002,1003,1004,1005 -d 1000 > raw/utillie_dcgm.log
#           1001 GR_ENGINE_ACTIVE  1002 SM_ACTIVE  1003 SM_OCCUPANCY
#           1004 PIPE_TENSOR_ACTIVE            1005 DRAM_ACTIVE
```

Representative `dcgmi dmon` shape alongside `nvidia-smi`:

```
#Entity  GRACT   SMACT   SMOCC   TENSO   DRAMA          nvidia-smi GPU-Util
GPU 0    0.982   0.008   0.001   0.000   0.002                    99 %
GPU 0    0.987   0.008   0.001   0.000   0.002                   100 %
GPU 0    0.981   0.008   0.001   0.000   0.003                   100 %
```

**The one-paragraph explanation your report needs:** `GPU-Util` is the fraction of sample intervals during which *at least one kernel was resident*. One thread on one SM of 132 satisfies that condition perfectly, so the field reads ~100%. `SM_ACTIVE` averages residency across all SMs and reads ~0.008 — consistent with 1 of 132 SMs busy. `PIPE_TENSOR_ACTIVE` reads 0.000 because no HMMA instruction ever issues. **`GPU-Util` answers "is something running"; it never answered "how much of the machine is working," and it is not a rounding error away from doing so — it is a different question.** This is the credibility exhibit in your report: it is the evidence that every other number you present was chosen to measure useful work.

#### D — model load and the fit check

Take L3/L5's arithmetic and confirm it against reality, because the gap between predicted and actual is where "it fits" turns into an OOM at 3 a.m.

```bash
python - <<'PY'
import torch
free0, total = torch.cuda.mem_get_info()
print(f"free before: {free0/2**30:.1f} GiB of {total/2**30:.1f} GiB")
PY

# then, in the serving stack of your choice, load the FP8 checkpoint and:
nvidia-smi --query-gpu=memory.used,memory.total --format=csv
```

```
  PREDICTED (L5 §10)                        MEASURED
  weights  70e9 × 1 B      = 70.0 GB        memory.used after load: 74.8 GiB
  CUDA context + allocator ≈  1.0 GB        (of 79.6 GiB total)
  framework workspace      ≈  3.0 GB
  ─────────────────────────────────────
  total                    ≈ 74.0 GB        Δ = +0.8 GiB  ✔ model is sound

  remaining for KV cache = 79.6 − 74.8 = 4.8 GiB
  at 0.156 MiB/token (Llama-3-70B FP8 KV, L6 §Worked example):
      4.8 × 1024 / 0.156 = ~31,500 tokens of cache
      ≈ 15 concurrent requests at 2k context   ← this is your batch ceiling
```

Write down the predicted-vs-measured delta. A report that says "predicted 74.0 GB, measured 74.8 GiB, delta +1%" is a report whose author checked; one that only states the prediction is a report whose author assumed.

#### E — decode throughput and the batch curve

Serve the model, then benchmark it. Using vLLM's built-in benchmark CLI:

```bash
# terminal 1 — serve
vllm serve <model-id> \
    --quantization fp8 \
    --max-model-len 4096 \
    --gpu-memory-utilization 0.92 \
    --disable-log-requests

# terminal 2 — single stream first: this is the BANDWIDTH CEILING check
vllm bench serve --model <model-id> \
    --dataset-name random --random-input-len 1024 --random-output-len 512 \
    --num-prompts 20 --max-concurrency 1

# terminal 2 — then the batch curve
for C in 1 2 4 8 16 32 64 128 256; do
  echo "=== concurrency $C ==="
  vllm bench serve --model <model-id> \
      --dataset-name random --random-input-len 1024 --random-output-len 512 \
      --num-prompts $((C * 8)) --max-concurrency $C
done | tee raw/batch_curve.log

# terminal 3 — telemetry for the whole run
dcgmi dmon -e 1001,1002,1004,1005 -d 1000 > raw/serve_dcgm.log
```

`vllm bench serve` prints a summary block; the two lines that matter are the throughput lines and the latency percentiles:

```
============ Serving Benchmark Result ============
Successful requests:                     128
Benchmark duration (s):                  61.42
Total input tokens:                      131072
Total generated tokens:                  65536
Request throughput (req/s):              2.08
Output token throughput (tok/s):         1067.05
Total Token throughput (tok/s):          3201.14
---------------Time to First Token----------------
Mean TTFT (ms):                          412.18
P99 TTFT (ms):                           988.40
-----Time per Output Token (excl. 1st token)------
Mean TPOT (ms):                          58.31
P99 TPOT (ms):                           74.02
==================================================
```

Three things to extract, and the trap to avoid:

1. **`Output token throughput` is your `tokens_per_sec`.** Not `Total Token throughput` — that includes prompt tokens, which are processed in the compute-bound prefill phase at a completely different rate. Using the total inflates your throughput and understates your cost, typically by 3× on a 1024-in/512-out mix. **Say which one you used.**
2. **The batch curve is the point.** Tabulate concurrency against output tok/s and P99 TPOT. Throughput rises steeply at first (each weight sweep is amortised over more sequences — L4), then flattens, then the KV cache fills and requests start queueing. Your operating point is **the largest concurrency that still meets your latency SLO**, not the peak of the curve.
3. **Check the single-stream number against the bandwidth-derived ceiling** from L4/L6:

```
  bandwidth-derived single-stream ceiling, H100 + 70B FP8:
      70 GB per weight sweep ÷ 2.61 TB/s MEASURED bandwidth = 26.8 ms/token
      → 37.3 tok/s theoretical maximum for one stream

  measured single-stream (from the --max-concurrency 1 run): 1000/58.31 = 17.2 tok/s
  → 46% of the bandwidth ceiling.
```

That 46% is a real, reportable finding: the gap is attention over the KV cache, sampling, the scheduler, and Python overhead — none of which the pure weight-streaming model accounts for. A report that states the ceiling, the measurement, and the ratio is doing analysis. One that states only the measurement is doing stenography.

#### F — the FP16 → FP8 delta, with all three savings computed

Repeat §5.D and §5.E at BF16/FP16 and diff. On an 80 GB H100 a 70B model at 16-bit does not fit at all, so you have two honest options: **(a)** use a smaller model (an 8B fits both ways comfortably) and report the delta at that scale, or **(b)** keep the 70B and report the FP16 path as TP=2 across two GPUs, which is the realistic production comparison but doubles your rental. Option (a) is the right call for a weekend budget; state which you did.

For an 8B model at 1024-in/512-out on one H100:

| | BF16 | FP8 | Delta |
|---|---|---|---|
| Weight bytes | 8e9 × 2 = 16.0 GB | 8e9 × 1 = 8.0 GB | **−50%** |
| `memory.used` after load | 18.9 GiB | 10.7 GiB | −43% (context/workspace don't shrink) |
| KV headroom (79.6 GiB total, 0.92 util) | 54.3 GiB | 62.5 GiB | +15% |
| Bytes per weight sweep | 16.0 GB | 8.0 GB | **−50%** |
| Single-stream tok/s | 92 | 141 | **1.53×** |
| Peak output tok/s (batch curve) | 4,820 | 7,510 | **1.56×** |
| GEMM-only multiple (§5.A) | — | — | 1.70× |

Now compute the bandwidth saving explicitly, because "half the bytes" is an abstraction until it has units:

```
  Bytes moved per decode step:  BF16 16.0 GB   →  FP8 8.0 GB   (Δ = 8.0 GB/step)

  At the FP8 peak of 7,510 output tok/s with an effective batch of B=128,
  the server performs 7510/128 ≈ 58.7 weight sweeps per second:

      bandwidth demand at BF16 = 16.0 GB × 58.7/s = 939 GB/s
      bandwidth demand at FP8  =  8.0 GB × 58.7/s = 470 GB/s
      HBM headroom reclaimed   = 469 GB/s  ( = 18% of the 2.61 TB/s measured
                                             ceiling, freed for KV traffic and
                                             therefore for a bigger batch )

  Over one rented hour: 469 GB/s × 3600 s = 1.69 PB of HBM traffic avoided.
```

And the accuracy side, so the trade is priced on both faces. The citable prior (L5 §14) is Neural Magic / Red Hat's FP8 Llama 3.1 quantisation: **86.55 vs 86.63 OpenLLM average, 99.91% recovery** on the fully-quantised 405B. State it as a trade:

```
  FP8 vs BF16, one H100, 8B model, measured this weekend:
      throughput   +56%     (4,820 → 7,510 output tok/s)
      weight bytes −50%     (16.0 → 8.0 GB)
      HBM traffic  −469 GB/s at the operating point
      accuracy     −0.09 pp expected (published prior; NOT measured here)
      → cost per token falls by 1 − 1/1.56 = 36% on identical hardware
```

If you did not measure accuracy, **say "not measured — flagged as open" and cite the prior.** Do not silently omit the one column that could reverse the decision.

### 6. Price — tier-aware, term-matched, dated

> **Dated snapshot — August 2026. Verify before you quote it.** GPU-cloud pricing moves fast enough that a six-month-old number is close to useless in a report. Re-pull everything below at build time from [`clustermax.semianalysis.com`](https://clustermax.semianalysis.com/) and [`gpu-index.semianalysis.com`](https://gpu-index.semianalysis.com/).

Every prior lesson treated `$/hr` as one number per SKU. It isn't, for two independent reasons.

**Reason 1: reliability tier.** SemiAnalysis's **ClusterMAX** rating system scores GPU-cloud providers across ten criteria — security, lifecycle, orchestration, storage, networking, reliability, monitoring, pricing, partnerships, availability — and buckets them into five tiers: **Platinum, Gold, Silver, Bronze, UnderPerform**. In the ClusterMAX 2.0 round, reaching Platinum required scoring roughly 90+ out of 100 in nearly every category, and only CoreWeave achieved it. Bronze is defined as "meets the minimum criteria and is still recommended," with the named common gaps being inconsistent support, subpar networking performance, unclear SLAs, and limited Kubernetes/Slurm integration. **The silicon is identical across tiers; what differs is whether it stays up, whether the interconnect delivers under load, and whether a ticket gets answered.**

**Reason 2: contract term.** The same GPU at the same provider is priced differently on-demand, spot, 1-month, 1-year, and multi-year. Spot H100 has been observed at roughly 41% below on-demand. Comparing an on-demand quote for one SKU against a reserved quote for another is the same category error as comparing across tiers.

The 2026 snapshot, with every figure carrying its basis:

| SKU | Basis | Observed range | Cohort median | Notes |
|---|---|---|---|---|
| H100 80GB | 1-yr reserved | **$1.45 – $2.99** (Bronze $1.45–2.00: SF Compute, Vast.ai, Hyperstack, RunPod Community; Silver $2.10–2.99: Lambda, AWS, GMI Cloud, Scaleway) | ~$3.15 across the whole rental cohort | The Bronze→Silver spread alone is >2× |
| H100 80GB | on-demand | **$2.19 – $11.06** | ~$4.17 dedicated clouds / ~$7.89 hyperscalers | The hyperscaler premium is roughly 1.9× |
| H200 141GB | reserved / listed | **$2.30 (FluidStack) – $13.78 (Azure)** | ~$4.11 | "H200 from $2.50/hr" is a representative Silver-tier reserved figure |
| B200 | on-demand | **$4.95 – $18.00** | — | Launch premium; wide and unstable |

Two market facts worth internalising alongside the table. First, the **H100 cohort median fell from more than $7/hr in early 2024 to roughly $3.15/hr** by this pass — a dollar figure without a date attached is not conservative, it is unverifiable. Second, the *same H100* spans roughly $0.80/hr (specialty spot) to well north of $90/hr (hyperscaler reserved multi-GPU bundles with everything attached). **"H100 costs $X/hr" is not a well-formed sentence.** Any $/hr figure you cite needs four fields or it isn't a number:

```
  $2.10/hr · H100 SXM · Silver tier · 1-year reserved · pulled 2026-08-15
   ▲         ▲           ▲            ▲                 ▲
   price     SKU         tier         contract term     date
```

For cross-checking a *serving* $/token figure against the public market, [Artificial Analysis](https://artificialanalysis.ai/) publishes per-model API pricing, but read its methodology first: it blends input and output prices at a fixed ratio (commonly **3:1 input:output**, i.e. `0.75 × input + 0.25 × output` per MTok; some views use a 7:2:1 cache-hit/input/output blend). A workload whose real ratio differs will see a materially different true cost. **Treat it as a sanity check on your own measured tok/s, never as a substitute for it.**

### 7. Assemble the arithmetic

Now put §5 and §6 together. Do it in this exact order so every intermediate is checkable.

```
  STEP 1 — normalise throughput to an hourly quantity
      tokens_per_hour = output_tok_per_sec × 3600

  STEP 2 — divide the hourly cost by the hourly output
      $/1M tokens = ($/hr) ÷ (tokens_per_hour / 1e6)

  STEP 3 — do it for both SKUs at a MATCHED tier and MATCHED term

  STEP 4 — state the decision rule as a ratio, not a difference
      SKU B is cheaper per token  ⇔  ($/hr ratio)  <  (tok/s ratio)

  STEP 5 — sensitivity: re-run steps 3–4 at a second matched tier and at a
      second contract term. Report which conclusions survive and which don't.
```

Fully worked, with units carried, using the §5 measurements and the §6 prices:

```
  ── H100, FP8 70B decode serving ──────────────────────────────────────────
     measured output throughput            = 2,000 tok/s      [MEASURE THIS]
     tokens_per_hour = 2000 × 3600         = 7,200,000 tok/hr
                                            = 7.2 Mtok/hr
     price: $2.10/hr · Silver · 1-yr reserved · 2026-08
     $/1M tokens = 2.10 / 7.2               = $0.2917 per million tokens

  ── H200, same model, same precision, same SLO ────────────────────────────
     measured output throughput            = 3,400 tok/s      [MEASURE THIS]
     tokens_per_hour = 3400 × 3600         = 12,240,000 tok/hr
                                            = 12.24 Mtok/hr
     price: $2.50/hr · Silver · 1-yr reserved · 2026-08
     $/1M tokens = 2.50 / 12.24             = $0.2043 per million tokens

  ── the ratio test ────────────────────────────────────────────────────────
     $/hr ratio   = 2.50 / 2.10 = 1.19×
     tok/s ratio  = 3400 / 2000 = 1.70×
     1.19 < 1.70  →  H200 is cheaper per token, by 1 − 0.2043/0.2917 = 30.0%

  ── and the compute-bound denominator, for the same box ───────────────────
     achieved FP8 GEMM (§5.A)              = 1,310 TFLOP/s = 1.31 PFLOP/s
     delivered per hour = 1.31 × 3600      = 4,716 PFLOP/hr
     $/useful-PFLOP = 2.10 / 4716           = $4.45e-4 per PFLOP
     (and MFU = 1310/1979 = 66.2%, so 33.8% of what you rented was not
      delivered as useful FLOPs — that gap is the engineering backlog)
```

**The 2,000 and 3,400 tok/s figures above are placeholders with the right shape, not measurements.** Substitute yours from §5.E; the arithmetic is what you are learning, and it is unchanged.

### 8. Sensitivity — the step that makes it defensible

One number is an assertion. A number plus its sensitivity is an argument. Re-run step 4 across the pricing surface:

| Comparison | H100 $/hr | H100 $/1M tok | H200 $/hr | H200 $/1M tok | Cheaper | Margin |
|---|---|---|---|---|---|---|
| **Matched, low tier, 1-yr reserved** | $2.10 | $0.2917 | $2.50 | $0.2043 | **H200** | 30% |
| **Matched, cohort median** | $3.15 | $0.4375 | $4.11 | $0.3358 | **H200** | 23% |
| **Matched, low end of each range** | $1.45 | $0.2014 | $2.30 | $0.1879 | **H200** | 7% |
| **Cross-tier: H100 cheap vs H200 hyperscaler** | $1.45 | $0.2014 | $13.78 | $1.1258 | **H100** | 5.6× |
| **Cross-tier: H100 hyperscaler vs H200 cheap** | $7.89 | $1.0958 | $2.30 | $0.1879 | **H200** | 5.8× |

**Read the table as two findings, not one.**

*Finding 1, robust:* at any **matched** tier and term, H200 wins on this workload, by 7–30%. The margin narrows as you move toward the cheap end (where H100's discounting is deeper), but the ranking never flips. That is a genuine hardware conclusion, and it traces directly to L6's mechanism: the H200 delivers 1.43× the bandwidth and 1.76× the capacity on identical compute silicon, and this workload is bound by both.

*Finding 2, a warning:* the moment you compare **across** tiers, the ranking flips — and by a margin (5.6×) far larger than the hardware difference (1.7×). A report that quotes "H100 $1.45/hr" (an easy-to-find Bronze headline) against "H200 $13.78/hr" (a hyperscaler list price, because that is what a search surfaced) has not measured a hardware comparison. **It has measured a tier comparison and drawn a hardware conclusion from it, and it would recommend the wrong SKU.**

The procedural fix is not cleverness, it is discipline: **state tier, provider, contract term, and date next to every dollar figure, and re-run the ranking at a second matched tier before trusting it.** If you can only get a negotiated rate for one SKU, get a comparable-basis quote for the other before concluding anything.

Two more sensitivities worth one line each in a real report:

- **Utilisation.** All of the above assumes the GPU is saturated. At 40% duty cycle every $/token figure rises by 2.5×, and the *ranking* can change if one SKU's advantage is capacity (which only pays off at high batch). State your assumed utilisation.
- **Throttling.** If §4's baseline or §5.A's clock log showed sustained throttling, your achieved throughput — and therefore your $/token — is a property of that rented card, not of the SKU. Say so.

### 9. From numbers to the document

The report is the deliverable, and it has one job: **let someone who did not run the benchmarks re-derive your conclusion.** That means every number appears with its provenance and every assumption is stated in one place. The structure that satisfies the module checkpoint:

```
  report.md
  ├─ 1. Conclusion (one paragraph, up front)
  │     "For <workload>, buy <SKU> at <precision>. $/1M tokens: <A> vs <B>
  │      at matched <tier>/<term>, measured <date>. Robust to tier; sensitive
  │      to utilisation below <X>%."
  ├─ 2. Assumptions table   (model, precision, in/out token mix, SLO,
  │                          assumed utilisation, pricing basis, date)
  ├─ 3. Achieved vs spec TFLOPS + roofline plot        ← §5.A, §5.B
  ├─ 4. The util-lie exhibit                           ← §5.C
  ├─ 5. Fit check: predicted vs measured HBM           ← §5.D
  ├─ 6. Decode ceiling + batch curve                   ← §5.E
  ├─ 7. FP16→FP8 delta (throughput/memory/bandwidth/accuracy)  ← §5.F
  ├─ 8. Cost arithmetic, shown step by step            ← §7
  ├─ 9. Sensitivity table                              ← §8
  └─ 10. Stack inventory + health baseline (appendix)  ← §4, L6
```

For the roofline plot in section 3, both axes are already measured:

```python
# roofline.py — uses YOUR measured ceilings, not the datasheet's
import matplotlib.pyplot as plt, numpy as np
PEAK_FLOPS = 1310e12      # §5.A, FP8, measured
PEAK_BW    = 2.61e12      # §5.B,       measured
ai = np.logspace(-1, 4, 400)
plt.loglog(ai, np.minimum(PEAK_BW * ai, PEAK_FLOPS), lw=2, label="measured roofline")
plt.axvline(PEAK_FLOPS / PEAK_BW, ls="--", label=f"ridge {PEAK_FLOPS/PEAK_BW:.0f} FLOP/B")
for name, x, y in [("8192³ GEMM (FP8)", 2731, 1310e12),
                   ("decode, batch 1",   0.5,   2.1e12),
                   ("decode, batch 128", 64,   1.8e13)]:
    plt.plot(x, y, "o"); plt.annotate(name, (x, y))
plt.xlabel("arithmetic intensity (FLOP/byte)"); plt.ylabel("achieved FLOP/s")
plt.legend(); plt.savefig("roofline.png", dpi=150)
```

(The GEMM's arithmetic intensity: `2N³ FLOP ÷ 3N² × 1 byte = 2N/3 = 5,461` at N=8192 for FP8 operands in the ideal case; use `2N/3 × (1/bytes_per_elem)` and state your assumption about cache reuse. Decode's is `~2 FLOP/byte ÷ batch`-scaled — L2's derivation.)

## Perspectives

**Finance/FinOps view.** Finance wants one number — $/1M tokens, or $/useful-PFLOP — that survives a quarterly review. The ClusterMAX tiering adds a complication finance doesn't intuitively expect: "cheaper $/hr" isn't even a clean single number without specifying the reliability tier and the contract term, so a finance-facing report has to carry both as first-class fields next to the price, not footnotes. A report that says "H100: $2.50/hr" without a tier and a term is not more precise than one that says "H100: $1.45–2.99/hr, 1-yr reserved, Bronze through Silver, pulled 2026-08" — it is *less* precise, dressed up as more precise.

**Engineering view.** Engineering supplies the inputs finance cannot produce alone: achieved TFLOPS against the *dense* spec, achieved bandwidth utilisation, measured output tok/s at a stated batch size and SLO, and the predicted-vs-measured HBM delta. None of that is negotiable or vendor-dependent — it is what you measured on the box. The discipline is refusing to let a spec-sheet number substitute for a measured one anywhere in the chain, and refusing to let `Total Token throughput` stand in for `Output token throughput` because it makes the number look better.

**Vendor/market view.** SemiAnalysis built ClusterMAX *because* the market is not homogeneous — the same silicon, rented from different providers, is not the same product once you account for networking quality under load, support responsiveness, and hardware-validation cadence (the same fleet-observability discipline CoreWeave's HPC Verification framework runs hourly against idle nodes, and the reason CoreWeave was the sole Platinum in the 2.0 round). Tiering is a market-structure fact, not a pricing gimmick: a Bronze box and a Platinum box running the identical H100 SXM chip are genuinely different purchases.

**Failure-mode view.** The thesis — "per-hour intuition gets it backwards" — describes the most common mistake a junior platform engineer makes in a SKU review, and tier-blindness is its most common *specific* form in 2026: comparing a cherry-picked cheap quote for SKU A against a rack-rate for SKU B and mistaking the gap for a hardware conclusion. §8's table shows that error producing a 5.6× "margin" on a 1.7× hardware difference. The fix is procedural, not clever.

## Real-world use cases

- **[SemiAnalysis, "The GPU Cloud ClusterMAX Rating System — How to Rent GPUs"](https://newsletter.semianalysis.com/p/the-gpu-cloud-clustermax-rating-system-how-to-rent-gpus) and the [ClusterMAX 2.0 update](https://newsletter.semianalysis.com/p/clustermax-20-the-industry-standard).** The industry-standard framework for exactly the "which SKU, from which vendor, at what tier" decision this capstone asks you to make. What it shows: raw $/hr comparison across GPU clouds is misleading without a reliability tier attached; the five tiers (Platinum/Gold/Silver/Bronze/UnderPerform) are scored across ten named criteria, and Platinum required ~90+/100 in nearly every one — only CoreWeave cleared it in the 2.0 round.
- **[SemiAnalysis, "How Much Do GPU Clusters Really Cost?"](https://newsletter.semianalysis.com/p/how-much-do-gpu-clusters-really-cost).** What it shows: the market-level pricing trend — the H100 cohort median falling from >$7/hr in early 2024 to roughly $3.15/hr — which is the concrete justification for this lesson's "date every dollar figure" rule. A number that moved 55% in two years will not survive being quoted undated.
- **[Google Cloud (Medium), "What Does 4.4% GPU Utilization Actually Mean?"](https://medium.com/google-cloud/what-does-4-4-gpu-utilization-actually-mean-ee61fabebbf0).** A real B200 production benchmark (96×B200, GKE Autopilot, ~1M tok/s aggregate) reporting 4.4% FLOPS utilisation and 1.5% tensor-active at decode batch=1 — while `GPU-Util` would read near 100%. This is L1's util-lie and L2's roofline landing in one dated, dollar-adjacent number, and it is the published proof for §1's "single-digit compute-MFU at decode is *correct*" claim.
- **[Databricks/MosaicML, "LLM Inference Performance Engineering: Best Practices"](https://www.databricks.com/blog/llm-inference-performance-engineering-best-practices).** A production team running multiple serving backends (vLLM, TensorRT-LLM, FasterTransformer) grounding the prefill-compute-bound / decode-memory-bound framing that §1's choice-of-denominator rule depends on — practitioner voice behind the math.
- **[CoreWeave HPC Verification](https://docs.coreweave.com/docs/platform/fleet-management/hpc-verification).** An hourly, per-idle-node hardware-validation framework running health checks against nodes that are not currently rented. What it shows: what a high-tier GPU-cloud rating actually buys — the tier premium in §6's table is partly the cost of somebody else running L6's Xid/throttle/fabric checks continuously so your job doesn't land on a degraded card.

## Worked example — H100 vs H200, 70B serving, and why the tier you compare matters

**Workload (stated assumptions — copy these verbatim into the report's section 2):**

- Model: Llama-3-70B class, decode-heavy chat serving, ~1k-token prompts, ~512-token generations.
- Precision: FP8 weights (~70 GB) on both SKUs, single GPU each — the clean comparison, no tensor-parallel tax on either side.
- Operating point: the largest concurrency that holds the p95 TPOT SLO, from §5.E's batch curve.
- Assumed utilisation: 100% (saturated). Flagged as a sensitivity in §8.
- Throughput: **H100 2,000 tok/s output, H200 3,400 tok/s output.** *These are illustrative placeholders with the right shape — substitute your own §5.E measurements.*
- Hardware facts (L6 §1, the crux): both SKUs run **1,979 FP8 dense TFLOPS and 989.5 BF16 dense TFLOPS — identical**, because H200 is the same GH100 die. H100 has 80 GB HBM3 at 3,350 GB/s; H200 has 141 GB HBM3e at 4,800 GB/s. **H200 buys zero extra compute.** It buys 1.43× bandwidth and 1.76× capacity — and decode is bound by both, so both convert to throughput.
- Why the throughput ratio is ~1.7× and not ~1.43×: bandwidth supplies 1.43× directly, and the extra 61 GB of capacity supplies ~11× the KV headroom (L6 §Worked example), which allows a much larger batch and therefore amortises each weight sweep over more sequences. The two effects compose until the compute ceiling or the SLO stops them.

**Pricing (2026-08 snapshot; re-verify before citing):** H100 SXM 1-yr reserved from **$2.10/hr** (Silver-tier representative), Bronze floor **$1.45/hr**, cohort median **~$3.15/hr**, hyperscaler on-demand up to **$11.06/hr**. H200 141GB reserved from **$2.30/hr** (FluidStack, low end of the observed range) with a **~$2.50/hr** Silver-tier representative, cohort median **~$4.11/hr**, and a top of range at **$13.78/hr** (Microsoft Azure).

```
  $/1M-tokens = ($/hr) ÷ (output_tok/s × 3600 / 1e6)

  H100 hourly output = 2000 × 3600 / 1e6 =  7.20 Mtok/hr
  H200 hourly output = 3400 × 3600 / 1e6 = 12.24 Mtok/hr
```

| Comparison | Basis | H100 $/hr | H100 $/1M tok | H200 $/hr | H200 $/1M tok | Cheaper | Margin |
|---|---|---|---|---|---|---|---|
| Matched — low end of each range | 1-yr reserved | $1.45 | $0.2014 | $2.30 | $0.1879 | **H200** | 7% |
| Matched — Silver representative | 1-yr reserved | $2.10 | $0.2917 | $2.50 | $0.2043 | **H200** | 30% |
| Matched — cohort median | mixed cohort | $3.15 | $0.4375 | $4.11 | $0.3358 | **H200** | 23% |
| **Cross-tier** — H100 floor vs H200 hyperscaler | mismatched | $1.45 | $0.2014 | $13.78 | $1.1258 | *H100* | *5.6×* |
| **Cross-tier** — H100 hyperscaler vs H200 floor | mismatched | $7.89 | $1.0958 | $2.30 | $0.1879 | *H200* | *5.8×* |

Worked longhand for one row, so the arithmetic is auditable:

```
  H100, Silver, 1-yr reserved:
      $2.10/hr ÷ 7.20 Mtok/hr = $0.29166.../Mtok  →  $0.2917 per million tokens
  H200, Silver, 1-yr reserved:
      $2.50/hr ÷ 12.24 Mtok/hr = $0.20424.../Mtok →  $0.2043 per million tokens
  saving = 1 − (0.2043 / 0.2917) = 0.2997 → 30.0% cheaper per token on H200
  cross-check with the ratio rule: ($/hr ratio 1.19) < (tok/s ratio 1.70) ✔
```

**The pedagogical point, stated plainly:** at **matched** tier and term, H200 wins by 7–30% — a stable ranking driven by the bandwidth and capacity math, holding across every matched pair in the table. The moment you compare **across** tiers or terms, the ranking flips and the apparent margin (5.6×) dwarfs the real hardware difference (1.7×). The comparison stopped being about silicon and became about procurement, while still wearing a hardware costume.

**Recommendation (defended):** *For this 70B FP8 decode-serving workload, buy H200 and serve FP8, holding pricing tier and contract term constant across both SKUs. At matched bases the H200's 141 GB unlocks the batch size the Hopper compute engine can already sustain but the 80 GB H100 starves, and its 1.43× bandwidth lifts the bound path directly, for a 7–30% cost-per-token advantage depending on tier. The conclusion is robust to tier choice and to contract term, and it is sensitive to sustained utilisation — below roughly 50% duty cycle the capacity advantage stops paying and the comparison narrows. State the tier, provider, term, and date next to every dollar figure. If only one SKU's discounted quote is available, obtain a comparable-basis quote for the other before drawing a conclusion — never compare a negotiated rate against a rack rate.*

## Practice — THE MODULE DELIVERABLE

This lesson's practice **is** the [GPU Efficiency & Cost Report](../practice/gpu-efficiency-report/README.md). Execute §3's timeline, then assemble §9's document. Concretely:

1. **Run the weekend plan (§3–§5)** on one rented H100 or H200. Keep every raw log in `raw/` — `gemm.log`, `clocks_during_gemm.csv`, `utillie_smi.csv`, `utillie_dcgm.log`, `batch_curve.log`, `serve_dcgm.log`. **The raw logs are part of the deliverable**; a conclusion whose evidence was deleted is an opinion.
2. **Achieved-vs-spec TFLOPS + roofline (L1–L2, §5.A–B).** Your measured GEMM against the **dense** spec, the resulting MFU, achieved bandwidth against HBM spec, and the roofline plot built from *your measured axes* with at least three workload points on it.
3. **FP16→FP8 delta (L5, §5.F).** Measured throughput multiple, memory delta, the bandwidth-saving arithmetic in GB/s and PB/hr, and the accuracy line (measured, or the published prior with an explicit "not measured" flag). State whether your multiple is throughput-only or compounded with a footprint change.
4. **The utilisation-lies demo (L1, §5.C).** `GPU-Util` ≈ 100% beside `SM_ACTIVE` ≈ 0.008 and `PIPE_TENSOR_ACTIVE` ≈ 0.000, timestamped, with the one-paragraph mechanism explanation.
5. **Decode ceiling + batch curve (L3–L4, §5.D–E).** Predicted-vs-measured HBM footprint; single-stream tok/s against the bandwidth-derived ceiling with the ratio stated; the concurrency sweep table with P99 TPOT, and the point where KV capacity saturates it.
6. **Cost-per-useful-work + SKU recommendation (this lesson, §6–§8).** The `$/1M-tokens` comparison of at least two SKUs from measured throughput and **dated, tier-and-term-labelled** `$/hr`, the arithmetic shown longhand for at least one row, the sensitivity table, and a defended buy call that names what the conclusion is robust to and what it is sensitive to.
7. **Stack inventory appendix (L6, §4).** Versions, driver branch and EOL, Xid baseline, throttle baseline, power limit.

Acceptance = the module [checkpoint](../checkpoint.md). The report is done when every checkpoint box ticks and you can defend each number — including its pricing tier, term, and date — from memory.

## Common pitfalls

1. **Tier-blind or term-blind price comparison.** Comparing a Bronze reserved quote for one SKU against a hyperscaler on-demand rate for another and treating the gap as a hardware conclusion. §8's table shows this producing a 5.6× "margin" on a 1.7× real difference, with the ranking reversed.
2. **Dividing by the sparse spec.** H100's 3,958 FP8 / 1,979 BF16 headline figures include 2:4 structured sparsity; dense peaks are exactly half. Using the sparse number halves your reported MFU and doubles every $/PFLOP downstream — and it errs in the direction nobody questions.
3. **Reporting `Total Token throughput` as your tokens/sec.** It includes prompt tokens processed during compute-bound prefill. On a 1024-in/512-out mix it is roughly 3× the output throughput, so it understates cost per generated token by 3×. Use `Output token throughput` and say so.
4. **Reporting compute-MFU for a decode workload and calling it bad.** Decode is memory-bound; single-digit compute-MFU is the *correct* reading, as the 96×B200 production benchmark's 4.4% shows. Report bandwidth utilisation there instead.
5. **Single-point, undated pricing.** "The" H100 price does not exist: $1.45–2.99/hr reserved across Bronze and Silver, ~$3.15/hr cohort median, up to $11.06/hr on-demand at a hyperscaler, and a cohort median that fell 55% in two years. Cite a range, a tier, a term, and a date.
6. **Treating public $/token benchmarks as directly reproducible.** Artificial Analysis and similar aggregators blend input and output prices at a fixed ratio (commonly 3:1); a workload with a different real ratio sees a different true cost. Sanity check, not substitute.
7. **Conflating a throughput-only multiple with a compounded one.** Databricks' ~1.5× realised FP8 *training* throughput (fixed hardware footprint) and a serving cost improvement that also removes a GPU are both real and both correctly cited — but quoting one while implying the other's scope is easy and wrong. State explicitly what is held fixed.
8. **Benchmarking on a throttled or under-provisioned card and blaming the software.** Check `clocks_throttle_reasons` and `power.limit` at the gate (§4) and again during the sweep. A 500 W-capped H100 has a genuinely lower roofline, and "78% of spec" on that card may be "95% of achievable."
9. **Leaving the instance running while you write the report.** §6 onward needs no GPU. This is the one pitfall in the list that costs money directly rather than credibility.

## Self-check

- **For the stated 70B FP8 decode workload, is H100 or H200 cheaper per token? Does the answer depend on anything besides the hardware?** *Answer:* At matched tier and term, H200, by 7–30%. Longhand at the Silver representative: H100 `$2.10/hr ÷ (2000 × 3600/1e6 = 7.20 Mtok/hr) = $0.2917/Mtok`; H200 `$2.50/hr ÷ (3400 × 3600/1e6 = 12.24 Mtok/hr) = $0.2043/Mtok` → 30% cheaper. The ratio rule confirms it: `$/hr ratio 1.19 < tok/s ratio 1.70`. The ranking holds at every matched basis because the H200's 1.43× bandwidth and 1.76× capacity compose to ~1.7× throughput on identical compute silicon, while the tier premium applies roughly proportionally to both SKUs. But yes, it depends on more than hardware: comparing *across* tiers (H100 floor $1.45 vs H200 Azure $13.78) flips the ranking to H100 by 5.6× — a pricing artifact, not a hardware fact. It is also sensitive to sustained utilisation, since the capacity advantage only pays at high batch.
- **Why can't you meaningfully say "H100 costs $X/hr" in 2026?** *Answer:* Two independent axes. **Reliability tier** — ClusterMAX scores providers across ten criteria (security, lifecycle, orchestration, storage, networking, reliability, monitoring, pricing, partnerships, availability) into Platinum/Gold/Silver/Bronze/UnderPerform; 1-yr-reserved H100 runs $1.45–2.00 at Bronze and $2.10–2.99 at Silver for *identical silicon*, and Platinum required ~90+/100 across nearly every category. **Contract term** — on-demand H100 spans $2.19–11.06 with medians of ~$4.17 (dedicated clouds) and ~$7.89 (hyperscalers), and spot has been observed ~41% below on-demand. Plus time: the cohort median fell from >$7/hr in early 2024 to ~$3.15/hr. So a price needs four fields — price, tier, term, date — or it is not a number.
- **Where did `nvidia-smi GPU-Util` mislead in your report, and which metric corrected it?** *Answer:* In the util-lie exhibit, a single-block kernel doing no useful math drove `GPU-Util` to ~100% because a kernel was resident on every sample interval, while `DCGM_FI_PROF_SM_ACTIVE` read ~0.008 (about 1 of 132 SMs) and `PIPE_TENSOR_ACTIVE` read 0.000 (no HMMA ever issued). The same pattern appears in production: the 96×B200 benchmark reports 4.4% FLOPS utilisation and 1.5% tensor-active while GPU-Util would read near 100%. The corrective metrics are **achieved bandwidth utilisation** (achieved GB/s ÷ HBM spec) for a memory-bound workload's ceiling and **aggregate output tok/s** for useful work delivered. `GPU-Util` measures residency; it never measured FLOPs or tokens.
- **Walk the full arithmetic from a measured `vllm bench serve` run to a $/1M-token figure.** *Answer:* (1) Take **`Output token throughput (tok/s)`** from the summary block — not `Total Token throughput`, which includes prefill and would understate cost roughly 3× on a 1024-in/512-out mix. (2) `tokens_per_hour = tok/s × 3600`. (3) `$/1M tokens = ($/hr) ÷ (tokens_per_hour / 1e6)`. (4) Attach the four price fields (tier, provider, term, date). (5) Compare SKUs with the ratio rule: cheaper per token iff `($/hr ratio) < (tok/s ratio)`. (6) Re-run at a second matched tier and term and report which conclusions survive. Worked: 2,000 tok/s → 7.20 Mtok/hr; at $2.10/hr → $0.2917 per million tokens.
- **Your FP8-vs-FP16 estimate, with assumptions — and which multiple are you quoting?** *Answer:* Measured on one H100 with an 8B model: weights 16.0 → 8.0 GB (−50%), peak output throughput 4,820 → 7,510 tok/s (**1.56×**), raw GEMM multiple 1.70×, HBM demand at the operating point 939 → 470 GB/s (469 GB/s reclaimed, ≈1.69 PB/hr). Cost per token therefore falls `1 − 1/1.56 = 36%` **on identical hardware** — this is a *throughput-only* multiple. Quoting a larger figure (2–3×) requires also folding in a *footprint* change (e.g. 70B dropping from TP=2 to a single GPU), which is a different question. Databricks' published ~1.5× realised FP8 *training* throughput is the throughput-only kind at fixed footprint; state which you mean before quoting. Accuracy: expect ≈0.1 pp on published FP8 PTQ recovery (86.55 vs 86.63 OpenLLM average, 99.91%), flagged as not independently measured unless you measured it.
- **What does the ClusterMAX rating actually score, beyond price, and why does it exist?** *Answer:* Ten criteria — security, lifecycle, orchestration, storage, networking, reliability, monitoring, pricing, partnerships, availability — composited into five tiers. It exists because the same GPU silicon rented from different providers is not a fungible product once you account for how reliably it stays up, how the interconnect performs under load, and how fast a ticket is answered. Concretely, part of what a top-tier premium buys is somebody running fleet health validation continuously — the CoreWeave HPC Verification framework health-checks idle nodes hourly, which is L6's Xid/throttle/fabric discipline as a purchased service rather than a thing you do yourself at 2 a.m.
- **You measured 772 TFLOPS BF16 on an H100 while the clock log shows a sustained 1,650 MHz. What do you report?** *Answer:* Both denominators, labelled. Against spec: `772 / 989.5 = 78.0%`. Against what the card could actually deliver at its observed clock: peak `= 132 SM × 4096 FLOP/SM/clk × 1.65e9 = 892 TFLOPS`, so `772 / 892 = 86.5%`. The first answers "am I getting the silicon I rented" (a procurement question — the card is throttling, and that is worth raising with the provider); the second answers "how good is my kernel" (an engineering question). Reporting only one of them lets a reader draw the wrong conclusion about which team owns the gap.

## Connections & what's next

This lesson is the module's convergence point: L1's util-lie is the discipline that keeps your throughput numbers honest and supplies the report's credibility exhibit; L2's roofline decides *which* denominator ($/PFLOP or $/1M-tokens) is even meaningful for the workload; L3–L4's memory arithmetic is where the tok/s ceiling and the batch curve come from; L5's precision lever is the largest throughput multiplier available without new silicon, and its `SMs × FLOP/SM/clock × clock` identity is what lets you correct a throttled measurement rather than misattribute it; L6's generational table and stack hygiene is what you are choosing *between*, and the gate that makes any measurement trustworthy. None of those stand alone as interview answers. This one is where they become a single defensible number with a stated sensitivity.

It also opens the next two modules directly. **Module 04 (GPU on Kubernetes)** takes the cost-attribution instinct built here and makes it *per-pod* and *live* — your `gpu-cost-operator` needs exactly this $/useful-work framing, but computed continuously against a running fleet instead of a weekend benchmark, which means the measurements in §5 become exported metrics rather than log files. **Module 11 (GPU cost and unit economics)** — the program's signature module — goes further: allocated-vs-utilised cost, $/token and $/run as organisational metrics, and the FOCUS cost-schema standard, all resting on the literacy this capstone builds. The pricing discipline you just practised — never quote a dollar figure without its tier, term, and date — is the same discipline module 11 demands at fleet scale rather than benchmark scale.

There is no next lesson in this module — the [checkpoint](../checkpoint.md) is next. Ship the [GPU Efficiency & Cost Report](../practice/gpu-efficiency-report/README.md), answer the checkpoint's depth probes cold, and flip this module's status.

## References & further reading

**Primary sources**
- SemiAnalysis, [ClusterMAX GPU cloud ratings](https://clustermax.semianalysis.com/) — the live tiering system (Platinum/Gold/Silver/Bronze/UnderPerform) and the ten scoring criteria this lesson's §6 is built from; **time-sensitive, re-check the current tier bounds and provider assignments before quoting.**
- SemiAnalysis, [GPU rental price index](https://gpu-index.semianalysis.com/) — live $/hr across SKUs and contract terms (on-demand through multi-year); **time-sensitive, refresh every number before you cite it.**
- [Artificial Analysis](https://artificialanalysis.ai/) and its [methodology page](https://artificialanalysis.ai/methodology) — read for the blended input:output price weighting (commonly 3:1, i.e. `0.75×input + 0.25×output` per MTok; some views use a 7:2:1 cache/input/output blend) before treating any public $/token figure as comparable to your own measurement.
- [NVIDIA H100 Tensor Core GPU Architecture whitepaper](https://resources.nvidia.com/en-us-tensor-core/gtc22-whitepaper-hopper) — the canonical **dense** FP8 (1,979 TFLOPS) and BF16 (989.5 TFLOPS) figures behind §5.A's MFU math, and the 132-SM / 1,830 MHz configuration behind the throttling correction.
- [NVIDIA H200 datasheet / product page](https://www.nvidia.com/en-us/data-center/h200/) — canonical 141 GB / 4.8 TB/s figures behind the Worked example's hardware table, and the confirmation that H200 shares H100's compute die.
- [NVIDIA DCGM documentation](https://docs.nvidia.com/datacenter/dcgm/latest/user-guide/feature-overview.html) — the `DCGM_FI_PROF_*` field definitions and `dcgmi dmon -e` syntax used throughout §5 (1001 GR_ENGINE_ACTIVE, 1002 SM_ACTIVE, 1003 SM_OCCUPANCY, 1004 PIPE_TENSOR_ACTIVE, 1005 DRAM_ACTIVE).
- [vLLM benchmarking CLI documentation](https://docs.vllm.ai/en/latest/benchmarking/cli/) — `vllm bench serve` / `vllm bench throughput`, the `--max-concurrency` and `--request-rate` semantics, and the definitions of the reported metrics (including the `Output` vs `Total` token throughput distinction that §5.E turns on).

**Real-world engineering blogs**
- SemiAnalysis, ["The GPU Cloud ClusterMAX Rating System — How to Rent GPUs"](https://newsletter.semianalysis.com/p/the-gpu-cloud-clustermax-rating-system-how-to-rent-gpus) and ["ClusterMAX 2.0: The Industry Standard GPU Cloud Rating System"](https://newsletter.semianalysis.com/p/clustermax-20-the-industry-standard) — why raw $/hr comparison across neoclouds is misleading without a reliability tier, and the tier definitions and scoring thresholds quoted in §6.
- SemiAnalysis, ["How Much Do GPU Clusters Really Cost?"](https://newsletter.semianalysis.com/p/how-much-do-gpu-clusters-really-cost) — the market-level price collapse (>$7/hr early 2024 → ~$3.15/hr cohort median) that motivates dating every dollar figure.
- Databricks/MosaicML, ["LLM Inference Performance Engineering: Best Practices"](https://www.databricks.com/blog/llm-inference-performance-engineering-best-practices) — a production multi-backend serving team's prefill/decode cost framing, underlying §1's choice-of-denominator rule.
- Databricks/MosaicML, ["Turbocharged Training: Optimizing the Databricks Mosaic AI Stack With FP8"](https://www.databricks.com/blog/turbocharged-training-optimizing-databricks-mosaic-ai-stack-fp8) — the real ~1.5× realised FP8 throughput at >50% MFU, the throughput-only multiple this lesson contrasts against a compounded serving figure.
- CoreWeave, [HPC Verification docs](https://docs.coreweave.com/docs/platform/fleet-management/hpc-verification) — the hourly, per-idle-node hardware-validation framework that concretises what a high-tier GPU-cloud rating is actually buying.
- Google Cloud (Medium), ["What Does 4.4% GPU Utilization Actually Mean?"](https://medium.com/google-cloud/what-does-4-4-gpu-utilization-actually-mean-ee61fabebbf0) — a dated B200 production benchmark (96×B200, ~1M tok/s aggregate, 4.4% FLOPS utilisation, 1.5% tensor-active) tying L1's util-lie, L2's roofline, and this lesson's cost math into one number.

**Deeper dives**
- Horace He, ["Making Deep Learning Go Brrrr From First Principles"](https://horace.io/brrr_intro.html) — the module's conceptual anchor; the compute/memory/overhead trichotomy behind every roofline placement this lesson prices.
- Google DeepMind, ["How to Scale Your Model" — "All About Rooflines"](https://jax-ml.github.io/scaling-book/roofline/) — a modern (2025), free, frontier-lab-authored complement to the 2009 roofline paper, with a GPU-specific chapter; useful for extending §9's plot to multi-GPU and communication rooflines.
- Imbue, ["From bare metal to a 70B model: infrastructure set-up and scripts"](https://imbue.com/research/70b-infrastructure/) — a warts-and-all account of training at scale; grounds the "reliability is part of the cost" argument behind §6's tiering in a concrete engineering narrative.
