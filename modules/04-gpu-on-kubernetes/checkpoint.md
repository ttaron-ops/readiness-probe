# 📦 Checkpoint — 04 · GPU on Kubernetes

The **completion gate** — the hardest-probed operational module. Prove it with the
[Per-pod GPU attribution](practice/per-pod-attribution/) exporter + failure-mode log,
and answer the probes cold. You've passed when you can, **unaided**:

## Pass criteria

- [ ] **1 · Install from scratch.** Bring up the GPU Operator on a fresh single-node
      cluster and get a CUDA pod running.
- [ ] **2 · Diagnose crash-loops.** Fix a crash-looping driver pod and a non-registering
      device plugin **from logs alone**.
- [ ] **3 · Upgrade a fleet.** Run a driver upgrade through the Operator's state machine
      and describe a 200-node rollout (drain, maxParallel, rollback).
- [ ] **4 · Device-plugin lifecycle.** Narrate Register / ListAndWatch / Allocate and
      explain **why there is no fractional GPU** — cold.
- [ ] **5 · Sharing modes.** Explain **MIG vs time-slicing vs MPS** — mechanism,
      isolation, and when — cold, with the **cost-attribution consequence** of each.
- [ ] **6 · Attribution, live.** Map a device UUID to a pod/namespace via the
      pod-resources API, and explain why a **time-sliced UUID breaks 1:1 attribution**.
- [ ] **7 · DRA.** Install the NVIDIA DRA driver and schedule a GPU via a `ResourceClaim`
      on a supported k8s version (noting the 1.34.0/.1 caveat).
- [ ] **8 · Ship it.** Produce the **per-pod attribution exporter** and the
      **GPU Operator failure-mode log**.

## Depth probes (answer cold)

- [ ] Put the GPU Operator init-container dependency chain in order; which component configures containerd?
- [ ] GPU pods Pending, `nvidia.com/gpu` = 0 — walk your diagnosis.
- [ ] "CUDA driver version insufficient" *inside* a container — root cause?
- [ ] `NVIDIA_VISIBLE_DEVICES` vs `NVIDIA_DRIVER_CAPABILITIES` — what does each control?
- [ ] Why must you drain a node before reconfiguring MIG?
- [ ] Why does a neighbor OOM you under time-slicing, and why can't you bill from allocation?
- [ ] What signal *can* you attribute time-sliced GPU usage with, and from where?
- [ ] `ResourceClaim` vs a device-plugin request — what's structurally better for attribution?

## Interview-readiness proxy

- [ ] You maintain a written **failure-mode log** of every GPU Operator break you hit and fixed.
- [ ] You can turn a device UUID + utilisation into a per-pod dollar figure and say what you joined to get there.

## Answers / notes

_Record answers as you close each lesson; link the exporter code + failure-mode log for items 1–8._
