---
lesson: "08.6"
title: "Job orchestration"
module: "08"
concept: "Job orchestration"
status: not-started
est_time: "6h"
prev: "05-failure-and-elasticity.md"
next: "07-data-pipeline.md"
artifacts: []
sources: 12
---
# 08.6 · Job orchestration
> **Concept.** A training job on Kubernetes is a *set* of pods that must find each other and agree on a world; the Training Operator is the controller that turns one `PyTorchJob` object into master/worker pods and injects the rendezvous env — the glue between 06's placed gang and torchrun's process group.
>
> Module: [🧮 08 — Distributed training infrastructure](../README.md) · Deliverable: [Survive-a-failure lab](../practice/survive-a-failure/README.md)

## Where this fits

05 taught you the causal chain that keeps a running job alive: XID/DCGM signal → drain →
re-rendezvous → resume. This lesson shows you the Kubernetes object that chain lives
inside — the CRD and controller that create the pods, wire the rendezvous env, and decide
what "restart the pod" even means, as distinct from torchrun's own worker-process
restart. Next, 07 leaves failure/orchestration behind and turns to a different way a GPU
wastes money — the input pipeline starving it — using the same SM%/cost lens this module
has been building since 03.

## Why this matters

You know how to run a distributed job by hand: `torchrun --nnodes=... --rdzv-endpoint=$HEAD:29400`
on every node, with `MASTER_ADDR`, `RANK`, and `WORLD_SIZE` set correctly. That works for
two nodes you SSH into. It does not work when a researcher submits forty jobs a day, each
wanting 8–256 GPUs, onto a shared cluster you operate — because now *someone* has to
allocate the nodes, generate a unique rendezvous endpoint per job, set every pod's env
consistently, restart the pod set on failure, and tear it all down on completion. Doing
that by hand is the toil. **A CRD + controller is how Kubernetes removes toil: you
declare the desired job as one object and a controller reconciles the pods.** That
controller is the Kubeflow **Training Operator**, and its `PyTorchJob` custom resource is
the thing a training platform actually submits.

For the target role this is the crux of "I operate a training platform." The researcher
wants to write a training script and `kubectl apply` a job. You want gang scheduling,
topology-aware placement, quota, and clean failure semantics. The operator is the seam
where those two meet — and knowing *exactly* which env vars it injects, and where it
hands off to the scheduler you learned in module 06, is what separates "I've heard of
Kubeflow" from "I've debugged a PyTorchJob that wouldn't rendezvous."

## What's new here (building on module 05, module 06, 08.5)

- **Module 06 owns placement; this lesson owns expression + rendezvous wiring.** In
  module 06 you learned gang scheduling (all-or-nothing co-scheduling) and topology-aware
  placement. That answers *where* the pods land. It does **not** answer how the pods,
  once landed, discover each other and form a process group. That is the operator's job,
  and it is what 08.6 adds. The handoff: **module 06's scheduler places the gang; the
  operator hands those placed pods their rendezvous identity.** We do not re-teach gang
  scheduling — module 06 owns it.
- **08.5 owns the failure loop; this lesson owns the object it restarts.** 08.5's
  re-rendezvous happens *inside* a running pod set. Here you see what created that pod set
  and how a `restartPolicy` / `backoffLimit` on the CRD interacts with torchrun's own
  `--max-restarts` — two restart layers you must not double-count.
- **New:** the CRD → pods → env-injection mechanism, the master/worker distinction, and
  the direction of travel — **Trainer v2's unified `TrainJob`** replacing the per-framework
  operators.

## Core concepts

### The object: `PyTorchJob`

The Training Operator watches `PyTorchJob` (API group `kubeflow.org/v1`). You declare
two **replica types** — `Master` (exactly 1) and `Worker` (N) — each a pod template:

```yaml
apiVersion: kubeflow.org/v1
kind: PyTorchJob
metadata:
  name: ddp-demo
spec:
  nprocPerNode: "8"                 # workers (GPUs) per node → torchrun --nproc-per-node
  pytorchReplicaSpecs:
    Master:
      replicas: 1
      restartPolicy: OnFailure
      template:
        spec:
          containers:
          - name: pytorch
            image: my-registry/ddp:latest
            command: ["torchrun", "train.py", "--resume-from=/ckpt/latest"]
            resources: { limits: { nvidia.com/gpu: 8 } }
    Worker:
      replicas: 3                    # total world = (1 master + 3 workers) × nprocPerNode
      restartPolicy: OnFailure
      template:
        spec:
          schedulerName: scheduler-plugins-scheduler   # module 06: gang scheduler
          containers:
          - name: pytorch
            image: my-registry/ddp:latest
            command: ["torchrun", "train.py", "--resume-from=/ckpt/latest"]
            resources: { limits: { nvidia.com/gpu: 8 } }
```

The `Master` is not a parameter server — in DDP there is no central server. It is simply
the replica the operator designates as the **rendezvous host**: its (headless-service)
address becomes every worker's `MASTER_ADDR`. Master is rank 0's node.

### How the operator sets up the rendezvous env

This is the heart of the lesson. When the operator reconciles a `PyTorchJob`, for each
pod it **injects environment variables** so the containers form one process group with no
manual endpoint wiring:

- **`MASTER_ADDR`** — the DNS name of the master pod, backed by a **headless Service** the
  operator creates (e.g. `ddp-demo-master-0`). This is how workers find rank 0.
- **`MASTER_PORT`** — a fixed rendezvous port (default `23456`).
- **`WORLD_SIZE`** — total number of pods across Master + Worker replicas (the *node*
  count for the process group; `torchrun` then multiplies by `--nproc-per-node` for the
  process count).
- **`RANK`** — the pod's index: master is `0`, workers are `1..N`. Injected via the pod's
  ordinal so each pod gets a distinct rank.
- **`PYTHONUNBUFFERED`, `NCCL_*`** pass-through as configured.

The container's `torchrun` reads these and initializes the c10d rendezvous — you did *not*
write a `--rdzv-endpoint`; the operator's env + headless service supplied the equivalent.
So the mental model: **the operator is a rendezvous-env generator + Service creator; the
pod's torchrun is the consumer.** When a pod restarts (`restartPolicy: OnFailure`), it
comes back with the *same* `RANK`/`MASTER_ADDR`, so it rejoins the same world — which is
why the master's stable DNS name matters.

**Two restart layers, don't double-count:** the CRD's `restartPolicy`/`backoffLimit`
restarts *pods*; torchrun's `--max-restarts` (08.5) restarts *worker processes* within a
living pod set. Elastic re-rendezvous (08.5) is the torchrun layer; a pod crash-loop is
the operator layer. In practice you either lean on torchrun elasticity (fixed pods,
elastic processes) or on pod restarts — mixing both without understanding which is acting
produces confusing "it restarted but from the wrong step" incidents. See Common pitfalls
for the concrete failure mode.

### Where module 06 hands off

The `schedulerName: scheduler-plugins-scheduler` (or Volcano, or Kueue admitting to a
gang-capable scheduler) line is the seam. The flow:

```
  kubectl apply PyTorchJob
        │
        ▼
  Training Operator ── creates Master+Worker pods, headless Service, injects env
        │                (pods are still Pending — not yet placed)
        ▼
  [mod.06] gang scheduler ── admits ALL pods together or none (gang), honoring
        │                     topology-aware placement (same rail/NVLink domain)
        ▼
  pods bind to nodes ── kubelet starts containers
        │
        ▼
  torchrun in each pod reads MASTER_ADDR/RANK/WORLD_SIZE ── rendezvous ── training
```

So **module 06 places the gang; the operator expressed and wired it; torchrun forms the
world.** The operator does not itself do gang scheduling — it emits pods with the right
`schedulerName` and (for PodGroup-based schedulers) the PodGroup/gang annotations, then
module 06's scheduler enforces all-or-nothing. If gang scheduling is *absent*, you get
the classic failure: master + 2 of 3 workers schedule, the third waits for a node, and
the scheduled pods sit burning quota blocked in rendezvous — the exact partial-deadlock
module 06 exists to prevent.

### The direction: Trainer v2's `TrainJob`

Historically Kubeflow shipped a **per-framework** zoo: `PyTorchJob`, `TFJob`,
`MPIJob`, `XGBoostJob`, each its own CRD and controller quirks. **Kubeflow Trainer v2
unifies these into a single `TrainJob` API.** PyTorch's own blog announced Kubeflow
Trainer joining the PyTorch ecosystem alongside v2.0's GA in July 2025; v2.2 (March
2026) added JAX and XGBoost runtimes, Flux Framework/HPC integration, TrainJob
observability, and — per the project's own release notes — full compatibility with the
Training Operator v1 CRDs (`PyTorchJob`, `MPIJob`, `JAXJob`, `XGBoostJob`), unifying them
under `TrainJob` while smoothing migration. Two ideas worth carrying:

- **One `TrainJob` + `TrainingRuntime`/`ClusterTrainingRuntime`.** Instead of framework
  CRDs, a `TrainJob` references a reusable *runtime* (a blueprint: the pod templates,
  the ML framework, gang policy). Platform teams curate runtimes; researchers submit a
  small `TrainJob` against one. Separation of platform concern from user concern — the
  same pattern you value in every other CRD you operate.
- **Built on JobSet + Kueue-native.** v2 uses upstream **JobSet** for the pod-group
  lifecycle and integrates **Kueue** for queueing/quota (module 06's admission layer)
  rather than each operator reinventing it.

**What to say in an interview:** *"We run PyTorchJob today because that's what's deployed
and battle-tested; the direction is Trainer v2's `TrainJob`, which unifies the
per-framework operators behind one API and a reusable runtime, built on JobSet + Kueue."*
That is exactly the deployed-reality-plus-direction framing the role wants.

### The target-company pattern: CoreWeave SUNK (Slurm-on-Kubernetes)

Big-tech GPU shops have a cultural split: **researchers want Slurm** (`sbatch`,
`srun`, array jobs, the HPC muscle memory), **platform wants Kubernetes** (declarative,
reconciled, one control plane for everything). CoreWeave's **SUNK** (Slurm on Kubernetes)
resolves it by running **Slurm *as* workloads on Kubernetes**: `slurmctld` and `slurmd`
run in pods, nodes are shared with K8s scheduling, so researchers get the Slurm CLI and
job semantics they want while the platform keeps K8s discipline (one scheduler substrate,
GPU operator, observability). It scales to >100k GPUs with individual jobs >32k GPUs.

The lesson for you: **PyTorchJob/TrainJob and SUNK are two answers to the same question —
"how do I express a gang training job on shared GPU infra."** Kubeflow answers it
K8s-native; SUNK answers it by bringing Slurm's UX onto K8s. Know both; a CoreWeave
interview will probe the Slurm-on-K8s seam, an Anthropic/K8s-native shop the operator seam.

## Perspectives

**Platform-team view.** The CRD+controller pattern is toil elimination — the operator's
job is narrowly "generate rendezvous env + create Service." Understanding that *narrow*
scope is what lets you correctly attribute a stuck job to the operator layer vs the
scheduler layer vs the torchrun layer during an incident, rather than guessing.

**Researcher/user view.** The researcher wants `kubectl apply` or `sbatch` and nothing
else. SUNK's whole value proposition is giving Slurm-native researchers that experience
without giving up K8s-native platform discipline underneath — a genuine cultural tension
(HPC muscle memory vs cloud-native reconciliation loops), not just a technical one. A
third-party (non-CoreWeave) treatment of this same tension: SkyPilot's
["Slurm vs K8s for AI Infra"](https://blog.skypilot.co/slurm-vs-k8s/), which notes even
frontier labs split — Meta trains on Slurm-flavored HPC clusters at tens of thousands of
GPUs while OpenAI scaled Kubernetes to thousands of nodes for GPT-3 — so this isn't a
CoreWeave-only narrative.

**API-evolution view.** The per-framework CRD zoo — `PyTorchJob`/`TFJob`/`MPIJob`/
`XGBoostJob` — unifying into `TrainJob` is a specific, dateable migration: v2.0 GA July
2025, v2.2 gains PyTorchJob-compatibility March 2026. Teach it as a **live migration in
progress**, not a done deal — `PyTorchJob` remains what's actually deployed at most shops
today, and the v2.2 compatibility layer exists precisely because that migration isn't
finished.

**Two-scheduler-layers view.** The "don't double-count restarts" point — CRD
`restartPolicy`/`backoffLimit` vs torchrun `--max-restarts` — is a real, easy-to-get-wrong
operational distinction. It's worth treating as its own perspective because it's exactly
where the 08.5 ↔ 08.6 seam bites in a live incident: was that restart the operator
recreating a crashed pod, or torchrun re-rendezvousing a shrunk world? The log evidence
looks similar; the correct next action does not.

## Real-world use cases

- **CoreWeave — SUNK (Slurm on Kubernetes)**
  ([blog](https://www.coreweave.com/blog/sunk-slurm-on-kubernetes-implementations),
  [docs](https://docs.coreweave.com/products/sunk),
  [SC23/SLUG23 slide deck](https://slurm.schedmd.com/SLUG23/CoreWeave-SLUG23.pdf)).
  `slurmctld`/`slurmd` running as pods, scaling to >100k GPUs with individual jobs
  >32k GPUs. What it shows: the exact target-company pattern this module is calibrated
  to — a real production system with a public architecture writeup.
- **PyTorch — "PyTorch on Kubernetes: Kubeflow Trainer Joins the PyTorch Ecosystem"**
  ([pytorch.org/blog](https://pytorch.org/blog/pytorch-on-kubernetes-kubeflow-trainer-joins-the-pytorch-ecosystem/),
  July 2025). What it shows: first-party PyTorch-team endorsement/integration for
  Kubeflow Trainer v2 — strong evidence that "TrainJob v2 is the direction" isn't just a
  Kubeflow-project claim.
- **Kubeflow — "Democratizing AI Model Training on Kubernetes: Introducing Kubeflow
  Trainer V2"** ([blog.kubeflow.org](https://blog.kubeflow.org/trainer/intro/)). What it
  shows: the project's own GA announcement (v2.0, July 2025) — primary-source
  confirmation of the exact version/date this lesson cites.
- **Kubeflow — Trainer v2.2 release notes**
  ([blog.kubeflow.org](https://blog.kubeflow.org/kubeflow-trainer-v2.2-release/), March
  2026). What it shows: JAX/XGBoost runtime support, Flux Framework/HPC integration,
  TrainJob observability, and full compatibility with Training Operator v1 CRDs
  (`PyTorchJob` included) — directly corroborates the exact version/date/PyTorchJob-
  compatibility claim this lesson makes.

## Worked example

Submit the 08.5 DDP job as a `PyTorchJob` (1 master + 3 workers, `nprocPerNode: 2` on a
CPU cluster for the demo). After `kubectl apply`:

1. `kubectl get pods -l training.kubeflow.org/job-name=ddp-demo` → `ddp-demo-master-0`,
   `ddp-demo-worker-0..2`.
2. `kubectl get svc` → the headless service `ddp-demo-master-0` (this is `MASTER_ADDR`).
3. Inspect a worker's injected env:
   ```bash
   kubectl exec ddp-demo-worker-1 -- env | grep -E 'MASTER_ADDR|MASTER_PORT|RANK|WORLD_SIZE'
   # MASTER_ADDR=ddp-demo-master-0
   # MASTER_PORT=23456
   # WORLD_SIZE=4          # 1 master + 3 workers (pods)
   # RANK=2                # worker-1 → pod rank 2
   ```
4. `kubectl logs ddp-demo-master-0` → torchrun rendezvous lines showing all 4 ranks
   joined, then training steps. **You wrote no endpoint** — the operator + headless
   service supplied the rendezvous coordinates, and the gang scheduler (module 06)
   admitted all four pods together.

That `env | grep` output — proof the operator wired the rendezvous — is the artifact.

## Practice — feeds the deliverable (optional stretch)

See [`../practice/survive-a-failure/README.md`](../practice/survive-a-failure/README.md)
for the full deliverable spec. **Goal:** submit a PyTorchJob on a local cluster and prove
how the rendezvous env is set.

1. **Cluster:** `kind create cluster` (or minikube). CPU-only is fine — rendezvous logic
   is identical; NCCL becomes Gloo, which is exactly what you want for a laptop demo.
2. **Install the Training Operator:**
   `kubectl apply -k "github.com/kubeflow/training-operator/manifests/overlays/standalone?ref=v1.8.1"`
   (or the Trainer v2 manifests if exploring the direction). Confirm the `PyTorchJob` CRD:
   `kubectl get crd pytorchjobs.kubeflow.org`.
3. **Submit** the DDP job from 08.5 as a `PyTorchJob` (1 master + N workers, small
   `nprocPerNode`, `command: ["torchrun", "train.py"]`, backend gloo).
4. **Inspect the rendezvous:** `kubectl exec <worker-pod> -- env | grep -E
   'MASTER_ADDR|MASTER_PORT|RANK|WORLD_SIZE'` and `kubectl get svc` for the headless
   master service. Capture the master's torchrun log showing all ranks joined.

**Acceptance (deliverable, optional stretch):** a submitted `PyTorchJob` reaching Running,
with the injected rendezvous env (`MASTER_ADDR`, `MASTER_PORT`, `RANK`, `WORLD_SIZE`)
captured from a worker pod and a one-line note mapping pod ordinal → `RANK`. Bonus:
delete a worker pod and observe the operator recreate it with the same rank (the pod-level
restart layer, distinct from 08.5's torchrun re-rendezvous).

## Common pitfalls

- **"The Training Operator does gang scheduling."** It doesn't. The operator creates pods
  and stamps `schedulerName`/PodGroup metadata; the *actual* all-or-nothing admission is
  done by module 06's separate gang scheduler (scheduler-plugins/Volcano/Kueue). Without a
  gang-aware scheduler configured, PyTorchJob alone will happily schedule a partial gang
  and deadlock it in rendezvous.
- **"PyTorchJob is deprecated, use TrainJob now."** As of the dates cited above,
  PyTorchJob is still the deployed/battle-tested reality at most shops. TrainJob v2 is
  GA (July 2025) and gaining PyTorchJob-compatibility (v2.2, March 2026), but it is a
  **migration in progress**, not a completed replacement — teach both, PyTorchJob as
  "what's running today."
- **"Master pod is a parameter server."** In DDP there's no central server — "Master" is
  just the operator's designation for the rendezvous-host replica. Conflating this with
  old-style parameter-server architectures is a common naming-induced confusion.
- **"SUNK replaces Kubernetes."** SUNK runs Slurm *as workloads on* Kubernetes — nodes and
  the scheduling substrate remain K8s; `slurmctld`/`slurmd` are themselves pods. It's an
  additive UX layer, not an alternative control plane.
- **"A pod restarting always means torchrun's elastic re-rendezvous kicked in."** A
  `restartPolicy: OnFailure` pod restart is the operator/K8s layer, separate from
  torchrun's own `--max-restarts` worker-process restart. Mixing them up without knowing
  which one fired produces exactly the "it restarted but from the wrong step" confusion —
  check whether the *pod* was recreated (new pod UID) vs the *process* inside a live pod
  restarted (same pod, new torchrun generation) before diagnosing further.

## Self-check

- **How does PyTorchJob set up the rendezvous environment for the workers?**
  **Answer:** The Training Operator reconciles the `PyTorchJob` into a master pod (rank 0)
  and N worker pods, creates a **headless Service** for the master, and **injects env vars**
  into every pod: `MASTER_ADDR` (the master's stable DNS name), `MASTER_PORT` (default
  23456), `WORLD_SIZE` (total pods), and `RANK` (pod ordinal: master 0, workers 1..N). The
  container's `torchrun`/`init_process_group` reads those and forms the c10d process group —
  no manual `--rdzv-endpoint` needed. The operator is the env generator + Service creator;
  torchrun is the consumer.

- **Where does module 06's gang scheduling hand off to the training operator?**
  **Answer:** The operator *creates* the master/worker pods (Pending) and stamps them with a
  `schedulerName` and PodGroup/gang metadata; **module 06's gang scheduler then admits all
  of them together or none**, honoring topology-aware placement, and binds them to nodes.
  So the operator expresses and wires the job; module 06 places it. If gang scheduling is
  missing, a subset of pods schedules and burns quota deadlocked in rendezvous — the
  partial-placement failure gang scheduling exists to prevent. Order: operator emits
  pods → module 06 gang-admits → kubelet starts → torchrun rendezvous.

- **What is Trainer v2's `TrainJob` unifying, and why?**
  **Answer:** It replaces the per-framework CRD zoo (`PyTorchJob`, `TFJob`, `MPIJob`,
  `XGBoostJob`) with **one `TrainJob` API** plus reusable `TrainingRuntime`/
  `ClusterTrainingRuntime` blueprints, built on upstream JobSet and Kueue. Why: eliminate
  duplicated per-framework controllers, give platform teams a curated-runtime abstraction
  separate from the researcher's small job spec, and standardize queueing/quota on Kueue.
  GA July 2025 (v2.0); v2.2 (March 2026) adds PyTorchJob compatibility for migration —
  so teach PyTorchJob as today's deployed reality, `TrainJob` as the direction.

- **A worker pod just came back up after a failure. How do you tell whether it was the
  operator's `restartPolicy` or torchrun's `--max-restarts` that fired?**
  **Answer:** Check the pod identity and the torchrun generation. If the operator's
  `restartPolicy: OnFailure` fired, Kubernetes recreated the *pod* — new pod start time /
  possibly new pod UID depending on controller, and the container's boot sequence runs
  from scratch. If torchrun's own elastic agent fired, the *pod* is unchanged (same pod,
  same start time) but the training process inside it exited and was relaunched by the
  agent with a bumped rendezvous generation and possibly a new `RANK`/`WORLD_SIZE` — visible
  in the torchrun logs, not in `kubectl get pods` timestamps. Conflating the two is exactly
  the "it restarted but from the wrong step" incident class.

## Connections & what's next

This lesson closes the loop 08.5 opened: the CRD and controller are the concrete
Kubernetes object underneath 08.5's "drain the pod" and "resume from checkpoint" arrows,
and the `schedulerName`/PodGroup handoff is where module 06's gang scheduling actually
plugs in. The two-restart-layers pitfall above is the single most common source of
confusing incidents at the 08.5/08.6 seam — know it cold for a debugging interview. Next,
07 leaves orchestration/failure behind for a different GPU-money problem: an
under-fed input pipeline starving an otherwise-healthy GPU, using the same SM%/cost
lens this module has used since 03.

## References & further reading

**Primary sources**
- Kubeflow — [Trainer docs](https://www.kubeflow.org/docs/components/trainer/) — TrainJob, runtimes, PyTorchJob migration, gang/Kueue integration.
- Kubeflow — [`kubeflow/trainer` repo](https://github.com/kubeflow/trainer) — manifests, CRDs, sample jobs; README confirms unified TrainJob positioning.
- Kubeflow — [PyTorchJob legacy-v1 user guide](https://www.kubeflow.org/docs/components/trainer/legacy-v1/user-guides/pytorch/) — the "PyTorchJob as deployed reality" reference doc.
- Kubeflow — [Migrating to Kubeflow Trainer v2](https://www.kubeflow.org/docs/components/trainer/operator-guides/migration/) — concrete PyTorchJob → TrainJob migration mechanics.
- CoreWeave — [SUNK docs](https://docs.coreweave.com/products/sunk) — architecture reference for the target-company pattern.

**Real-world engineering blogs**
- CoreWeave — ["A Slurm on Kubernetes Implementation for HPC and Large Scale AI"](https://www.coreweave.com/blog/sunk-slurm-on-kubernetes-implementations) and the [SLUG23 slide deck](https://slurm.schedmd.com/SLUG23/CoreWeave-SLUG23.pdf) — architecture diagrams, more detail than the blog post.
- PyTorch blog — ["PyTorch on Kubernetes: Kubeflow Trainer Joins the PyTorch Ecosystem"](https://pytorch.org/blog/pytorch-on-kubernetes-kubeflow-trainer-joins-the-pytorch-ecosystem/) (July 2025).
- Kubeflow blog — ["Democratizing AI Model Training on Kubernetes: Introducing Kubeflow Trainer V2"](https://blog.kubeflow.org/trainer/intro/) — v2.0 GA announcement.
- Kubeflow blog — [Trainer v2.2 release notes](https://blog.kubeflow.org/kubeflow-trainer-v2.2-release/) (March 2026) — JAX/XGBoost runtimes, Flux/HPC integration, TrainJob observability, PyTorchJob compatibility.

**Deeper dives**
- SkyPilot blog — ["Slurm vs K8s for AI Infra: Academic HPC vs Cloud-Native Reality"](https://blog.skypilot.co/slurm-vs-k8s/) — a third-party (non-CoreWeave) treatment of the same Slurm-vs-Kubernetes cultural split, useful for triangulating this isn't a CoreWeave-only narrative.
- > **Snapshot (2026-08).** Version/date claims above (v2.0 GA July 2025, v2.2 March 2026, SUNK's >100k-GPU figure) are dated snapshots against a fast-moving project — verify current state before quoting in a review.
