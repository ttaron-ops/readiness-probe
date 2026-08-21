---
lesson: "08.6"
title: "Job orchestration"
module: "08"
concept: "job-orchestration"
status: not-started
est_time: "6h"
prev: "05-failure-and-elasticity.md"
next: "07-data-pipeline.md"
artifacts: []
sources: 11
---
# 08.6 · Job orchestration
> **Concept.** A training job on Kubernetes is a *set* of pods that must find each other and agree on a world; the Training Operator is the controller that turns one `PyTorchJob` object into master/worker pods and injects the rendezvous env — the glue between 06's placed gang and torchrun's process group.
>
> Module: [🧮 08 — Distributed training infrastructure](../README.md) · Deliverable: [Survive-a-failure lab](../practice/survive-a-failure/README.md)

## Where this fits

08.5 taught you the causal chain that keeps a running job alive: XID/DCGM signal → drain →
re-rendezvous → resume. This lesson shows you the Kubernetes objects that chain lives
inside — the custom resources and controllers that create the pods, wire the rendezvous
environment, create the `PodGroup` the gang scheduler consumes, and decide what "restart"
even means at three different layers that all fire during the same incident. Next, 08.7
leaves failure and orchestration behind and turns to a different way a GPU wastes money —
the input pipeline starving it — using the same SM%/cost lens this module has been
building since 08.3.

Everything asserted below about controller behaviour was read out of source cloned during
this pass: `kubeflow/training-operator` at tag `v1.9.3` (the v1 `PyTorchJob` controller),
the same repository at `master` = **Kubeflow Trainer v2.3.0** (released 2026-08-05), and
`kubernetes-sigs/jobset` at `master`. Where a default is stated, it is the default in that
code.

## Why this matters

You already know how to run a distributed job by hand: `torchrun --nnodes=4
--nproc-per-node=8 --rdzv-backend=c10d --rdzv-endpoint=$HEAD:29400 train.py` on every node.
That works for two machines you SSH into. It stops working the moment a researcher submits
forty jobs a day, each wanting 8–256 GPUs, onto a cluster you operate — because now
*something* has to allocate nodes, generate a unique rendezvous endpoint per job, give every
pod a consistent view of the world size, create the gang object so the scheduler admits all
the pods or none, decide what happens when one pod dies, and garbage-collect the whole thing
on completion. Doing that by hand is the toil.

**A CRD plus a controller is how Kubernetes removes that toil: you declare the desired job
as one object and a controller reconciles it into the twenty other objects it implies.**
Knowing *which* twenty, and in what order, is the difference between "I've heard of
Kubeflow" and "I debugged a PyTorchJob that wouldn't rendezvous."

The specific failure this lesson prevents is the most common one in the whole module: a job
that is Running by every dashboard, has all its pods, has all its GPUs allocated, and is
doing nothing, because one pod's `WORLD_SIZE` disagrees with the others' and every rank is
blocked in `init_process_group` waiting for a rank that will never arrive. That job burns
the full gang rate until someone kills it. On 32 H100s at $3/GPU-hr that is $96/hr of
nothing.

## What's new here (calibration)

- **Module 06 owns admission; this lesson owns expression and wiring.** In module 06 you
  learned gang scheduling in mechanism-level detail: the `PodGroup` CRD, `minMember`, the
  `Permit` hold, the five extension points, Kueue's Workload-level alternative, and the
  native `Gang{minCount}` successor. **None of that is re-derived here.** What is new is
  who *creates* that `PodGroup`, what `minMember` it computes, and what happens to pod
  creation while the group is pending — which turns out to differ between the two gang
  backends the operator supports.
- **08.5 owns the elastic loop; this lesson owns the objects it runs inside.** 08.5's
  re-rendezvous happens *inside* a living pod set. Here you see what created that pod set,
  and how a `restartPolicy` on the replica spec, a `backoffLimit` on the run policy, and
  torchrun's `--max-restarts` form **three** independent restart layers whose interactions
  produce most of the confusing incidents at this seam.
- **Genuinely new:**
  - The complete `PyTorchJob` schema with every field that changes behaviour, and the exact
    environment the controller injects — including the `WORLD_SIZE` definition that the
    common summary of this API gets wrong.
  - **JobSet**, the primitive that replaced bespoke per-framework controllers: its
    `failurePolicy` rule engine (five actions, ordered matching), its three restart
    strategies, and the DNS model that makes rendezvous work without any env injection.
  - **Kubeflow Trainer v2's `TrainJob` + `TrainingRuntime`** as it actually ships in
    v2.3.0, including the `torch` plugin's env injection and the `coscheduling` plugin's
    `PodGroup` construction.
  - **Ray Train** as the same problem solved with actors and placement groups instead of
    pods and CRDs — the second orchestration idiom you will meet in a GPU shop.

## Core concepts

### 1. The problem: everything between `kubectl apply` and `init_process_group` returning

Before naming any API, write down what a distributed training job actually needs from its
orchestrator. A single-process job needs one thing: a pod. A `WORLD_SIZE`-way job needs
nine, and every one of them is a place it can hang:

| # | Requirement | What breaks if it is missing |
|---|---|---|
| 1 | N pods created with identical images/args | ranks run different code, collectives mismatch |
| 2 | A stable network identity per pod | rank 0 restarts, gets a new IP, nobody reconnects |
| 3 | Every pod agrees on `WORLD_SIZE` | all ranks block forever in rendezvous |
| 4 | Every pod has a distinct rank/index | two rank-0s, duplicated shards, silent wrong results |
| 5 | One rendezvous endpoint all pods can reach | ranks form two disjoint groups of size < N |
| 6 | All-or-nothing admission | partial gang holds GPUs and deadlocks (module 06) |
| 7 | A defined response to one pod dying | either a hung job or an infinite crash loop |
| 8 | A limit on retries | a poisoned job restarts forever, burning the gang |
| 9 | Cleanup on terminal state | dead pods hold quota nobody is using |

Requirements 1–5 are *rendezvous wiring*. Requirement 6 is module 06. Requirements 7–8 are
*failure policy* — the subject of half this lesson. Requirement 9 is why `runPolicy` exists.

Rendezvous itself is worth one paragraph of mechanism, because the failure modes above are
all shapes of it going wrong. PyTorch's `c10d` rendezvous is a *barrier with a store*.
Every participating agent connects to a `TCPStore` hosted by rank 0 (or to an external
etcd), writes its own identity into it, and waits until the number of participants that
have checked in equals the expected count. When the count is met, the store's contents are
frozen into a "round", each participant is assigned a rank, and every agent returns from
the barrier at the same time with a consistent `(rank, world_size)` mapping. Three things
follow directly: the expected count must be identical at every agent (or the barrier never
completes), rank 0's address must be reachable and stable (or nobody can check in), and a
new participant arriving later forces a *new round* — which is exactly the elastic
re-rendezvous of 08.5. An orchestrator's entire job, at the wiring level, is to make the
count agree and the address stable.

### 2. `PyTorchJob`, field by field

The v1 Training Operator watches `PyTorchJob` in API group `kubeflow.org/v1`. Here is a
complete object with every field that changes behaviour annotated. Defaults are from
`pkg/apis/kubeflow.org/v1/pytorch_defaults.go` and `common_types.go` at `v1.9.3`.

```yaml
apiVersion: kubeflow.org/v1
kind: PyTorchJob
metadata:
  name: ddp-demo
  namespace: research
spec:
  # ── PROCESSES PER POD ────────────────────────────────────────────────────────
  # Becomes PET_NPROC_PER_NODE, which torchrun reads as --nproc-per-node.
  # Accepted values: "auto" | "cpu" | "gpu" | an integer string.
  # DEFAULT: "auto" (DefaultNprocPerNode). "auto" means torchrun decides at runtime
  # from visible devices — which is why the operator CANNOT compute WORLD_SIZE
  # correctly when you leave it at the default. Always set it explicitly. See §4.
  nprocPerNode: "8"

  # ── RUN POLICY: lifecycle of the whole job ───────────────────────────────────
  runPolicy:
    # What to do with pods when the job reaches a terminal state.
    # All | Running | None.  DEFAULT: None  (pods are LEFT BEHIND — they hold
    # no GPU once terminated, but they do clutter the namespace and confuse
    # anyone reading `kubectl get pods`).
    cleanPodPolicy: Running
    # Delete the whole PyTorchJob this long after it finishes. DEFAULT: unset = never.
    ttlSecondsAfterFinished: 86400
    # Hard wall-clock cap measured from status.startTime. Unset = no cap.
    activeDeadlineSeconds: 172800
    # Retry budget for the JOB. Counted as the SUM of container restartCounts across
    # all Running pods of every replica type whose restartPolicy is OnFailure/Always/
    # ExitCode. Replica types set to Never are excluded from the count entirely.
    # (pkg/core/job.go: PastBackoffLimit)   DEFAULT: unset = unlimited.
    backoffLimit: 6
    # Create no pods at all while true. Kueue's webhook sets this to gate admission.
    # DEFAULT: false.
    suspend: false
    # Hand reconciliation to another controller. Only "" ,
    # "kubeflow.org/training-operator" (self) or "kueue.x-k8s.io/multikueue" are legal.
    # Immutable.
    managedBy: kubeflow.org/training-operator
    # ── GANG PARAMETERS (consumed by the operator to BUILD a PodGroup, §6) ─────
    schedulingPolicy:
      minAvailable: 4          # → PodGroup.spec.minMember.
                               # DEFAULT when omitted: the SUM of all replicas
                               # (here 1 + 3 = 4). Set it lower only for a job that
                               # can genuinely start short-handed.
      queue: research-queue    # Volcano queue name. Immutable. DEFAULT: "default".
      priorityClass: high      # Volcano PodGroup priorityClassName.
      scheduleTimeoutSeconds: 300   # → scheduler-plugins PodGroup field of the same
                                    # name; how long members wait at Permit.
      minResources:            # → PodGroup.spec.minResources. If omitted the operator
        nvidia.com/gpu: "32"   # COMPUTES it by summing minMember pods' requests.
        cpu: "128"
        memory: 1024Gi

  # ── ELASTIC POLICY: present ⇒ the pods run torchrun in elastic mode ──────────
  # Omit this block entirely for a fixed-size job.
  elasticPolicy:
    minReplicas: 2            # → the low end of PET_NNODES ("2:4")
    maxReplicas: 4            # → the high end
    maxRestarts: 3            # → PET_MAX_RESTARTS: torchrun agent restarts of the
                              #   worker group. THIS IS LAYER 3 OF THE RESTART STACK.
    rdzvBackend: c10d         # c10d | etcd | etcd-v2.  DEFAULT: c10d.
    rdzvPort: 29400
    # DEFAULT rdzv host when unset: "<jobname>-worker-0" — note WORKER-0, not master.
    # In elastic mode the rendezvous host is worker 0 and there is usually no Master
    # replica at all.
    rdzvId: ddp-demo-run-17
    rdzvConf:
      - key: timeout
        value: "900"

  # ── THE POD TEMPLATES ────────────────────────────────────────────────────────
  pytorchReplicaSpecs:
    Master:
      replicas: 1             # DEFAULT 1. Master is optional; if absent, no MASTER_ADDR/
                              # RANK/WORLD_SIZE are injected at all (§4).
      # Always | OnFailure | Never | ExitCode.  DEFAULT: OnFailure.
      # This value is COPIED INTO THE POD SPEC's restartPolicy, except for ExitCode
      # which becomes Never at the pod level (pkg/core/pod.go: SetRestartPolicy).
      # A restartPolicy set on the pod template itself is overwritten, and the
      # operator emits a SetPodTemplateRestartPolicy warning event when you try.
      restartPolicy: OnFailure
      template:
        spec:
          # Module 06's gang scheduler. If the operator was started with
          # --gang-scheduler-name and you ALSO set this, the operator warns
          # ("Another scheduler is specified…") and does not overwrite you.
          schedulerName: scheduler-plugins-scheduler
          containers:
            - name: pytorch          # THE NAME MATTERS. The controller finds the
                                     # rendezvous port by looking for a container
                                     # literally named "pytorch" with a port named
                                     # "pytorchjob-port". Rename it and env generation
                                     # fails with "port not found".
              image: registry.example.com/ddp:2026-08-14
              command: ["torchrun", "train.py", "--resume-from=/ckpt/latest"]
              ports:
                - name: pytorchjob-port   # DEFAULT injected if you omit it:
                  containerPort: 23456    # name pytorchjob-port, port 23456.
              resources:
                limits:   { nvidia.com/gpu: 8 }
                requests: { nvidia.com/gpu: 8 }
              volumeMounts:
                - { name: ckpt, mountPath: /ckpt }
                - { name: dshm, mountPath: /dev/shm }   # 08.7 will explain why.
          volumes:
            - name: ckpt
              persistentVolumeClaim: { claimName: ddp-demo-ckpt }
            - name: dshm
              emptyDir: { medium: Memory, sizeLimit: 32Gi }
    Worker:
      replicas: 3
      restartPolicy: ExitCode   # See §5: 1–127 = fail the job, ≥128 = retry the pod.
      template:
        spec:
          schedulerName: scheduler-plugins-scheduler
          containers:
            - name: pytorch
              image: registry.example.com/ddp:2026-08-14
              command: ["torchrun", "train.py", "--resume-from=/ckpt/latest"]
              ports:
                - { name: pytorchjob-port, containerPort: 23456 }
              resources:
                limits: { nvidia.com/gpu: 8 }
```

Two structural notes before the mechanism.

**`Master` is not a parameter server.** DDP has no central server; every rank holds a full
replica and gradients move by all-reduce (08.1–08.2). `Master` is purely the operator's
name for *the replica whose pod-0 DNS name becomes `MASTER_ADDR`*. Conflating it with
1990s parameter-server architectures is a naming accident that costs candidates points in
interviews.

**A job with only `Worker` replicas is legal and common.** The env-injection code is
guarded by `if spec.PyTorchReplicaSpecs[Master] != nil`. With no `Master`, the controller
injects **no** `MASTER_ADDR`, `MASTER_PORT`, `RANK`, or `WORLD_SIZE` — only
`PET_NPROC_PER_NODE` and `PET_NNODES` (plus the elastic `PET_RDZV_*` block if
`elasticPolicy` is set). That is not a bug: an elastic job discovers its own world through
the rendezvous backend, so a static `WORLD_SIZE` would be a lie.

### 3. What the controller actually does on one reconcile

```
  RECONCILE OF ONE PyTorchJob  (kubeflow/training-operator v1.9.3, common/job.go)
  ═══════════════════════════════════════════════════════════════════════════════

  PyTorchJob "ddp-demo" (1 Master + 3 Worker, nprocPerNode 8)
        │
        ▼
  ① terminal?  (Succeeded/Failed, past activeDeadlineSeconds, past backoffLimit)
        │  yes → DeletePodsAndServices per cleanPodPolicy
        │        → if gang enabled: DeletePodGroup   → set Failed/Succeeded  → STOP
        │  no
        ▼
  ② gang scheduling enabled?  (operator flag --gang-scheduler-name != "" && != None)
        │   DEFAULT IS "" — GANG IS OFF UNLESS THE OPERATOR WAS DEPLOYED WITH THE FLAG
        │
        ├─ yes ─▶ minMember  := runPolicy.schedulingPolicy.minAvailable
        │         (unset ⇒ Σ replicas = 4)
        │         minResources := schedulingPolicy.minResources
        │                         (unset ⇒ computed by summing minMember pods'
        │                          requests, honouring priority class ordering)
        │         SyncPodGroup(): create-or-update ONE PodGroup named after the job,
        │            owned by the job, in the job's namespace
        │              • backend "volcano"          → volcano.sh/v1beta1 PodGroup
        │                                             {minMember, queue, priorityClassName,
        │                                              minResources}
        │              • any other name             → scheduling.x-k8s.io/v1alpha1 PodGroup
        │                                             {minMember, minResources,
        │                                              scheduleTimeoutSeconds}
        │         ── THE FORK THAT SURPRISES PEOPLE ──
        │         DelayPodCreationDueToPodGroup(pg):
        │              volcano           → TRUE while pg.status.phase is "" or Pending
        │                                  ⇒ NO PODS ARE CREATED YET. Requeue.
        │              scheduler-plugins → always FALSE
        │                                  ⇒ pods are created immediately and wait
        │                                    at Permit holding assumed capacity.
        │
        └─ no ──▶ (skip; the pods will be handled by whatever scheduler they name)
        ▼
  ③ for each replica type, ReconcileServices():
        one HEADLESS Service per POD — name "<job>-<rtype>-<idx>", clusterIP None,
        selector on {job-name, replica-type, replica-index}, ports copied from the
        container's named ports.  Master AND workers each get one.
        ⇒ every pod is addressable at <job>-<rtype>-<idx>.<namespace>.svc.cluster.local
        ⇒ the name survives pod deletion; the IP behind it does not have to.
        ▼
  ④ for each replica type, ReconcilePods():
        for index in 0..replicas-1:
           • no pod?      → build template, setPodEnv(), SetRestartPolicy(),
                            DecoratePodTemplateSpec() [gang labels/annotations +
                            schedulerName], create with an ownerRef
           • pod Failed?  → apply the replica's restartPolicy (§5)
           • index out of range (you scaled down)? → delete the pod
        ▼
  ⑤ update status: per-type {active, succeeded, failed}, conditions
        Created → Running → (Restarting) → Succeeded | Failed
```

Read step ③ again, because it is the part most summaries get wrong. The operator creates a
**headless Service per pod**, not one Service for the master. `clusterIP: None` means the
Service publishes no virtual IP; DNS resolves the name straight to the pod's IP (A record),
and there is no kube-proxy hop. That gives you a stable *name* — `ddp-demo-master-0` — that
outlives any particular pod IP. That name, and only that name, is what `MASTER_ADDR`
contains.

### 4. The rendezvous environment, exactly

This is the highest-value paragraph in the lesson, because one of its rows is where people
get burned. From `pkg/controller.v1/pytorch/envvar.go`, `master.go` and `elastic.go` at
`v1.9.3`, every pod's container gets:

| Variable | Value the controller writes | Condition |
|---|---|---|
| `PYTHONUNBUFFERED` | `1` | always |
| `MASTER_ADDR` | `<job>-master-0` | only if a `Master` replica exists |
| `PET_MASTER_ADDR` | same as `MASTER_ADDR` | ″ |
| `MASTER_PORT` | the `pytorchjob-port` **container port of the Master** (default `23456`) | ″ |
| `PET_MASTER_PORT` | same | ″ |
| `WORLD_SIZE` | **`Σ replicas × nprocPerNode`** | ″ |
| `RANK` | master → `0`; worker *i* → `i + 1` | ″ |
| `PET_NODE_RANK` | same integer as `RANK` | ″ |
| `PET_NPROC_PER_NODE` | `spec.nprocPerNode` verbatim (string) | if `nprocPerNode` set |
| `PET_NNODES` | `Σ replicas` (non-elastic) or `"<min>:<max>"` (elastic) | always |
| `PET_RDZV_ENDPOINT` | `<rdzvHost or "<job>-worker-0">:<rdzvPort or worker port>` | elastic only |
| `PET_RDZV_BACKEND` | `elasticPolicy.rdzvBackend`, default `c10d` | elastic only |
| `PET_RDZV_ID`, `PET_RDZV_CONF`, `PET_STANDALONE` | as configured | elastic only |
| `PET_MAX_RESTARTS` | `elasticPolicy.maxRestarts` | elastic only |

**The row to memorise: `WORLD_SIZE` is a count of *processes*, not pods.** The code is
literally `worldSize := int(totalReplicas) * nprocPerNode`. For 1 master + 3 workers at
`nprocPerNode: "8"` the injected `WORLD_SIZE` is **32**, not 4. `PET_NNODES` is the pod
count, 4. If you have read a description of this API claiming `WORLD_SIZE` is the number of
pods, it is describing `PET_NNODES`.

There is a trap immediately behind that. `getNprocPerNodeInt()` returns **1** whenever
`spec.nprocPerNode` is not an integer string — and the *default* is `"auto"`. So a job that
omits `nprocPerNode` and lets torchrun auto-detect 8 GPUs per pod gets `WORLD_SIZE=4`
injected while torchrun actually launches 32 processes. If your training script reads
`os.environ["WORLD_SIZE"]` directly instead of letting torchrun overwrite it per-process,
you now have 32 processes convinced the world has 4 members. **Set `nprocPerNode` to an
explicit integer whenever anything downstream reads `WORLD_SIZE`.** (The reason this is not
a catastrophe in practice is that `torchrun` recomputes and re-exports `RANK`, `WORLD_SIZE`,
`LOCAL_RANK` per worker process from its own agent state; the injected values are inputs to
the agent, not to your script. But scripts that read them before `init_process_group` — or
that are launched with plain `python` instead of `torchrun` — see the wrong numbers.)

The `RANK` row has a similar subtlety worth naming: the operator writes the *node* index
into `RANK`, and separately into `PET_NODE_RANK`. Under torchrun, `PET_NODE_RANK` is the
one that matters (it becomes `--node-rank`), and each spawned worker's own `RANK` is
overwritten with `node_rank × nproc_per_node + local_rank`. `RANK` is set at the pod level
only so that a non-torchrun entrypoint (`python train.py` calling `init_process_group`
directly, one process per pod) still works.

```
  WHO WRITES WHAT, FOR ddp-demo (1 Master + 3 Worker, nprocPerNode 8)
  ══════════════════════════════════════════════════════════════════════════════

  operator writes into the POD                torchrun rewrites per PROCESS
  ────────────────────────────                ─────────────────────────────
  MASTER_ADDR=ddp-demo-master-0        ┐
  MASTER_PORT=23456                    │      MASTER_ADDR   (unchanged)
  WORLD_SIZE=32   ← Σreplicas × nproc  ├─────▶MASTER_PORT   (unchanged)
  RANK=2          ← pod ordinal        │      WORLD_SIZE=32 (agreed by rendezvous)
  PET_NODE_RANK=2                      │      RANK = 2×8 + local  → 16..23
  PET_NNODES=4    ← pod count          │      LOCAL_RANK = 0..7
  PET_NPROC_PER_NODE=8                 ┘      LOCAL_WORLD_SIZE = 8

  worker-1's pod holds ONE agent process that supervises EIGHT worker processes.
  The agent joins the c10d barrier once, on behalf of all eight.
```

### 5. Three restart layers, and how to tell which one fired

This is the operational core of the lesson. Three independent mechanisms can restart
something inside a PyTorchJob, and they compose in ways that surprise people.

```
  THE RESTART STACK, OUTERMOST FIRST
  ════════════════════════════════════════════════════════════════════════════════

  LAYER 1 — kubelet, container-level          governed by POD spec restartPolicy,
  ─────────────────────────────────           which the operator COPIES from
    container exits non-zero                  replicaSpec.restartPolicy
        │  restartPolicy: Always|OnFailure     (ExitCode ⇒ pod-level Never)
        ▼
    kubelet restarts the container IN PLACE
    same pod, same name, same node, same IP
    CrashLoopBackOff: 10s→20s→40s→…→5min cap
    container restartCount += 1
        │
        │  the pod NEVER reaches Failed, so the operator never sees it
        └─ but PastBackoffLimit() SUMS restartCount across running pods and
           fails the whole job once that sum exceeds runPolicy.backoffLimit

  LAYER 2 — the operator, pod-level           governed by replicaSpec.restartPolicy
  ─────────────────────────────────           evaluated only when pod.phase == Failed
    pod reaches Failed  (⇒ restartPolicy was Never or ExitCode)
        │
        ├─ OnFailure / Always            → operator DELETES the pod; next reconcile
        ├─ ExitCode && code ≥ 128        → recreates it with the SAME ordinal, SAME
        │                                  headless Service, SAME RANK
        │                                  job condition ← Restarting
        ├─ ExitCode && 1 ≤ code ≤ 127    → job condition ← Failed. No retry. Ever.
        └─ Never                         → nothing happens; the gang sits there

  LAYER 3 — the torchrun agent, process-level  governed by PET_MAX_RESTARTS
  ────────────────────────────────────────     (elasticPolicy.maxRestarts)
    one worker process dies inside a living pod
        │
        ▼
    the agent kills the remaining local workers, re-enters the c10d barrier,
    forms a NEW ROUND with whatever agents are present, and relaunches all
    workers from the last checkpoint your script loads.
    Pod UID unchanged. Pod start time unchanged. kubectl sees nothing.
```

**The `ExitCode` policy is the one worth knowing cold**, because it is the only one that
distinguishes "this job is broken" from "this node was broken". `IsRetryableExitCode` is
three lines: `return exitCode >= 128`. Unix reports a process killed by signal *N* as exit
code `128 + N`, so `137` is SIGKILL (the OOM killer, or a node drain), `143` is SIGTERM
(preemption, eviction), `139` is SIGSEGV. Those are environmental and worth retrying.
Codes 1–127 are your program calling `sys.exit(1)` — a shape error, a missing file, a bad
config. Retrying those just burns the gang N more times. So:

> `restartPolicy: ExitCode` on Worker replicas is the production setting for a job you
> care about. `OnFailure` retries a syntax error forever; `Never` leaves a gang stranded
> after a single node eviction.

Now the interaction that produces the confusing incident. With `restartPolicy: OnFailure`,
the operator copies `OnFailure` into the *pod spec*, so the **kubelet** restarts the
container in place and the pod's phase goes back to Running. The pod therefore almost never
reaches `Failed`, so the operator's Layer-2 branch almost never fires — and the only thing
that stops an endless crash loop is `PastBackoffLimit()` summing container `restartCount`
across the running pods. That means:

- **`backoffLimit` is a whole-job budget, not a per-pod budget.** With `backoffLimit: 6`
  and 32 worker pods, six restarts *anywhere in the job* — one pod restarting six times, or
  six pods restarting once each — fails the entire job.
- Replica types with `restartPolicy: Never` are **skipped in that sum entirely** (the code
  logs "not counted in backoff limit"). A job whose workers are `Never` and whose master is
  `OnFailure` has a `backoffLimit` that only counts master restarts.

Diagnosing which layer fired, during an incident, comes down to three observations:

| Evidence | Layer |
|---|---|
| `kubectl get pod -o jsonpath='{.status.containerStatuses[0].restartCount}'` incremented, pod age unchanged | 1 — kubelet |
| Pod **UID changed** (`kubectl get pod -o jsonpath='{.metadata.uid}'`), age reset, `Restarting` event on the PyTorchJob | 2 — operator |
| Pod UID *and* age unchanged, but the torchrun log shows a new rendezvous round and `[default] Starting worker group` | 3 — agent |

The event stream is the fastest tell: Layer 2 always emits `exitedWithCode` followed by
`PyTorchJobRestarting` on the *job* object. Layer 3 emits nothing to Kubernetes at all.

### 6. Where module 06 hands off — and what the operator does *not* do

Module 06 taught you what a gang scheduler does with a `PodGroup`. The handoff here is
narrow and precise:

```
  kubectl apply PyTorchJob
        │
        ▼
  Training Operator ── creates PodGroup{minMember, minResources}
        │            ── creates one headless Service per pod
        │            ── creates pods with schedulerName + PodGroup label/annotation
        │               (scheduler-plugins: label scheduling.x-k8s.io/pod-group=<job>
        │                volcano:           annotation scheduling.k8s.io/group-name=<job>)
        ▼
  [mod.06] gang scheduler ── admits all minMember pods together or none, honouring
        │                    topology-aware placement
        ▼
  kubelet starts containers
        ▼
  torchrun reads MASTER_ADDR / PET_* ── c10d barrier ── training
```

Three corrections to the folk version of this picture:

1. **The operator does create the `PodGroup`.** It is not merely stamping labels. It
   computes `minMember` (defaulting to the sum of all replicas), computes `minResources` if
   you did not supply it, sets an owner reference so the group is garbage-collected with
   the job, and reconciles the group on every pass.
2. **Gang scheduling is off by default.** `EnableGangScheduling()` returns
   `GangScheduling != "" && != "None"`, and the `--gang-scheduler-name` flag defaults to the
   empty string. A vanilla Training Operator install creates no `PodGroup` at all and your
   pods go to whatever scheduler they name. "We run PyTorchJob" is not the claim "we gang
   schedule."
3. **The two gang backends differ in what waits, and it matters for cost.** With Volcano,
   `DelayPodCreationDueToPodGroup` is true while the `PodGroup` phase is empty or Pending,
   so **the operator does not create pods until the group is admissible** — the queue holds
   an object, not capacity. With scheduler-plugins the same function returns `false`
   unconditionally, so all pods are created immediately and park at `Permit`, each holding
   an assumed node reservation for up to `scheduleTimeoutSeconds`. That is exactly the
   held-capacity-versus-held-nothing distinction module 06 drew between raw coscheduling
   and Kueue, appearing here as a controller-configuration choice.

### 7. JobSet: the primitive underneath everything newer

The per-framework operator zoo (`PyTorchJob`, `TFJob`, `MPIJob`, `XGBoostJob`) all
reimplemented the same three things: create N Jobs, give their pods stable DNS, restart the
set on failure. **JobSet** (`jobset.x-k8s.io/v1alpha2`, a Kubernetes SIG project) factors
that out into one CRD, and both Kubeflow Trainer v2 and several vendor stacks now build on
it rather than on bespoke controllers.

The model is one level up from pods: a JobSet contains **`replicatedJobs`**, each of which
is a `batch/v1` Job template instantiated `replicas` times. The child Jobs are named
`<jobset>-<replicatedJob>-<jobIndex>`, and their pods `…-<jobIndex>-<podIndex>`. That
naming is not cosmetic — it is the addressing scheme.

```yaml
apiVersion: jobset.x-k8s.io/v1alpha2
kind: JobSet
metadata:
  name: pretrain
spec:
  # ── DNS ──────────────────────────────────────────────────────────────────────
  network:
    enableDNSHostnames: true       # pods reachable at
                                   # <jobset>-<rj>-<jobIdx>-<podIdx>.<subdomain>
    subdomain: pretrain            # DEFAULT: the JobSet name.
    publishNotReadyAddresses: true # DEFAULT: true. CRITICAL for rendezvous: DNS
                                   # records exist BEFORE pods are Ready, so rank 0
                                   # is resolvable while it is still starting up.
  # ── WHEN IS THE SET DONE ─────────────────────────────────────────────────────
  successPolicy:
    operator: All                  # All | Any, optionally scoped to named
                                   # replicatedJobs via targetReplicatedJobs.
  # ── WHEN IS THE SET DEAD, AND WHAT HAPPENS FIRST ─────────────────────────────
  failurePolicy:
    maxRestarts: 20                # budget shared by every rule that "counts"
    restartStrategy: Recreate      # Recreate | BlockingRecreate | InPlaceRestart
    rules:                         # evaluated IN ORDER; FIRST MATCH WINS.
                                   # If no rule matches: action RestartJobSet.
      - name: HostMaintenance
        action: RestartJobSetAndIgnoreMaxRestarts   # does NOT consume the budget
        onJobFailureReasons: ["PodFailurePolicy"]   # a planned host event that the
                                                    # Job's own podFailurePolicy
                                                    # classified — infra, not us
      - name: OneBadNode
        action: RestartJob                          # recreate ONLY the failed child
        onJobFailureMessagePatterns: ["(?i)node .* not ready"]
        targetReplicatedJobs: ["workers"]
      - name: BadCode
        action: FailJobSet                          # stop immediately, no retry
        onJobFailureReasons: ["BackoffLimitExceeded"]
  ttlSecondsAfterFinished: 3600
  replicatedJobs:
    - name: workers
      replicas: 4                  # 4 child Jobs
      template:
        spec:
          parallelism: 8           # 8 pods per child Job …
          completions: 8           # … all of which must succeed
          backoffLimit: 0          # do NOT let batch/v1 retry a pod; let the
                                   # failurePolicy above decide. This is the normal
                                   # setting for training.
          template:
            spec:
              restartPolicy: Never
              containers:
                - name: trainer
                  image: registry.example.com/pretrain:2026-08-14
                  ports: [{ containerPort: 29500 }]
                  env:
                    - name: MASTER_ADDR
                      value: pretrain-workers-0-0.pretrain   # job 0, pod 0
                    - name: MASTER_PORT
                      value: "29500"
                    - name: NODE_RANK          # JobSet gives every pod its index
                      valueFrom:               # through the standard batch
                        fieldRef:              # completion-index annotation
                          fieldPath: metadata.annotations['batch.kubernetes.io/job-completion-index']
                  command:
                    - torchrun
                    - --nnodes=32
                    - --nproc-per-node=8
                    - --node-rank=$(NODE_RANK)
                    - --master-addr=$(MASTER_ADDR)
                    - --master-port=$(MASTER_PORT)
                    - train.py
                  resources:
                    limits: { nvidia.com/gpu: 8 }
```

Four mechanisms in that object are worth understanding rather than copying.

**The five failure-policy actions form a real decision table.** From
`api/jobset/v1alpha2/jobset_types.go`:

| Action | Scope of the restart | Counts against `maxRestarts`? | Use it for |
|---|---|---|---|
| `FailJobSet` | none — terminal | n/a | a failure your code caused |
| `RestartJobSet` | recreate **all** child Jobs | yes | the default; a synchronous job that cannot survive losing a rank |
| `RestartJobSetAndIgnoreMaxRestarts` | all child Jobs | **no** | known-benign infra churn (host maintenance, spot preemption) that must not exhaust the budget |
| `RestartJob` | only the failed child Job | yes | replica-parallel topologies where one DP replica can be rebuilt alone (this is the JobSet expression of the FT-HSDP idea from 08.5) |
| `RestartJobAndIgnoreMaxRestarts` | only the failed child Job | no | same, for benign causes |

Rules match on `onJobFailureReasons` (the Job condition reason) **and**
`onJobFailureMessagePatterns` (RE2 regexes over the message), are evaluated in list order,
and the first match executes. Both lists empty means "match anything". The status object
tracks four counters separately — `status.restarts`, `status.restartsCountTowardsMax`, and
per-child `jobRestarts[]` / `jobRestartsCountTowardsMax[]` — precisely so that the
ignore-max variants can be audited without inflating the budget. The budget check is on the
*sum*: `restartsCountTowardsMax + Σ jobRestartsCountTowardsMax[] < maxRestarts`.

**The three restart strategies differ in what they guarantee about the old pods.**
`Recreate` (the default) recreates each child Job as soon as its own predecessor's pods are
gone. `BlockingRecreate` waits until *every* pod from the previous iteration is deleted
before creating anything — which is what you want when the workload cannot tolerate a stale
rank still holding an NCCL communicator or a file lock. `InPlaceRestart` is the newest and
most interesting: healthy pods are **not deleted at all**; an agent container inside each
pod holds a barrier, and when the JobSet controller bumps
`status.currentInPlaceRestartAttempt` the agents release together and rerun the worker
process. Only genuinely failed pods are recreated. That removes pod scheduling, image pull,
and container start from the restart critical path — which, as 08.8 will price, is the term
that dominates cost at large scale. It is gated behind the `InPlaceRestart` feature gate and
requires `backoffLimit` at maximum on the child Jobs (the webhook enforces this).

**DNS replaces env injection.** With `enableDNSHostnames: true` JobSet creates a headless
Service named by `subdomain` and sets each pod's `hostname`/`subdomain`, so
`pretrain-workers-0-0.pretrain` resolves cluster-wide. You do not need a controller to tell
pod 17 where rank 0 lives; the name is derivable from the JobSet name and the indices.
`publishNotReadyAddresses: true` is what stops the classic startup race where every worker's
DNS lookup for the master `NXDOMAIN`s because the master pod is not Ready yet. The
`spec.coordinator` field formalises the same idea: name a `(replicatedJob, jobIndex,
podIndex)` and the controller stamps the resulting endpoint onto every pod as the
`jobset.sigs.k8s.io/coordinator` annotation.

**Gang scheduling is not JobSet's job.** JobSet has no `minMember`. It expects an external
admission layer — Kueue, or, going forward, the core Kubernetes workload-aware scheduling
APIs. The repo's own gang-scheduling example wires a `scheduling.k8s.io/v1alpha2`
`Workload` + `PodGroup` pair with `schedulingPolicy.gang.minCount: 6` and has the pod
template reference it via `spec.schedulingGroup.podGroupName` — the exact native mechanism
module 06 §10 previewed, here consumed by a JobSet. The `alpha.jobset.sigs.k8s.io/exclusive-topology`
annotation is the adjacent placement control: set it to a topology label key and every child
Job is scheduled exclusively within one topology domain (one rack, one TPU slice), which is
how you keep a tensor-parallel group inside one NVLink domain (08.1).

### 8. Kubeflow Trainer v2: `TrainJob` over a curated runtime

Kubeflow's answer to "one API for every framework" is `TrainJob` plus
`TrainingRuntime`/`ClusterTrainingRuntime`, and — this is the part that matters — **it
renders down to a JobSet.** The repository `kubeflow/training-operator` now *is* the
`kubeflow/trainer` repository; `VERSION` at `master` reads `v2.3.0`, released 2026-08-05.

The split is deliberate: the platform team publishes runtimes; the researcher submits a
small job against one.

```yaml
# ── PLATFORM TEAM OWNS THIS (cluster-scoped, reusable) ─────────────────────────
apiVersion: trainer.kubeflow.org/v1alpha1
kind: ClusterTrainingRuntime
metadata:
  name: torch-distributed
  labels:
    trainer.kubeflow.org/framework: torch
spec:
  mlPolicy:
    numNodes: 1
    torch: {}                    # selects the `torch` plugin → PET_* injection
  # optional: gang policy consumed by the coscheduling / volcano plugins
  # podGroupPolicy:
  #   coscheduling:
  #     scheduleTimeoutSeconds: 300
  template:                      # ← this is a JobSet spec, verbatim
    spec:
      replicatedJobs:
        - name: node
          template:
            metadata:
              labels:
                trainer.kubeflow.org/trainjob-ancestor-step: trainer
            spec:
              template:
                spec:
                  containers:
                    - name: node
                      image: pytorch/pytorch:2.13.0-cuda13.0-cudnn9-runtime
---
# ── RESEARCHER OWNS THIS ───────────────────────────────────────────────────────
apiVersion: trainer.kubeflow.org/v1alpha1
kind: TrainJob
metadata:
  name: pretrain-7b
  namespace: research
spec:
  runtimeRef:
    name: torch-distributed      # immutable
    apiGroup: trainer.kubeflow.org
    kind: ClusterTrainingRuntime
  trainer:
    numNodes: 4                  # overrides the runtime's numNodes
    numProcPerNode: 8
    image: registry.example.com/pretrain:2026-08-14
    command: ["torchrun", "train.py"]
    resourcesPerNode:
      limits: { nvidia.com/gpu: 8 }
```

What the controller does with that, from `pkg/runtime/framework/plugins/`:

- The **`torch` plugin** sets the trainer pod-set count from `numNodes`, resolves
  `numProcPerNode` (default `"auto"`; if `auto` and no GPU is requested it falls back to the
  CPU count), and injects `PET_NNODES`, `PET_NPROC_PER_NODE`, `PET_NODE_RANK` (from the
  batch completion-index field ref), plus `PET_MASTER_ADDR` = `<trainjob>-node-0-0.<trainjob>`
  and `PET_MASTER_PORT` = **29500**. Note the address form: it is the JobSet DNS name from
  §7, not an operator-managed Service per pod. It also appends container port 29500 so the
  headless Service exposes it.
- The **`jobset` plugin** renders the runtime template into an actual JobSet.
- The **`coscheduling` plugin** builds a scheduler-plugins `PodGroup` whose `minMember` is
  `Σ over pod-sets of count` and whose `minResources` is the summed per-pod requests times
  each set's count, with `scheduleTimeoutSeconds` from the runtime's `podGroupPolicy`. It
  only writes the group while the TrainJob is suspended — i.e. before Kueue admits it. A
  `volcano` plugin does the same for Volcano.
- `elasticPolicy` has **no v2 equivalent yet**: the torch plugin carries an explicit
  `TODO: Add support for PyTorch elastic when JobSet supports Elastic Jobs`. If you need
  elastic ranks today, that is a reason to stay on v1 `PyTorchJob` or to use Ray (§9).

**What to say in a review:** *"PyTorchJob is what is deployed and battle-tested; the
direction is Trainer v2's `TrainJob` over `TrainingRuntime`, which renders to JobSet and
integrates Kueue for admission. The one gap is elastic world size, which v2 does not model
yet."* That is deployed-reality-plus-direction, with the caveat that makes it credible.

### 9. Ray Train: the same nine requirements, solved with actors

A GPU shop that runs Ray solves §1's list without any CRD. `TorchTrainer` starts a
*controller actor* which requests a **placement group** of `num_workers` bundles, one per
GPU, then starts one worker actor per bundle, sets `MASTER_ADDR`/`RANK`/`WORLD_SIZE` from
the actors' own addresses, and calls your training function inside each.

```python
from ray.train import ScalingConfig, RunConfig, FailureConfig, CheckpointConfig
from ray.train.torch import TorchTrainer

trainer = TorchTrainer(
    train_loop_per_worker=train_func,          # plain PyTorch inside
    scaling_config=ScalingConfig(
        num_workers=32,                        # int, OR (min, max) for elastic
        use_gpu=True,                          # ⇒ 1 GPU per worker unless overridden
        resources_per_worker={"GPU": 1, "CPU": 12},
        placement_strategy="PACK",             # DEFAULT "PACK": bundles on as few
                                               # nodes as possible. "STRICT_SPREAD"
                                               # forces one bundle per node.
        accelerator_type="H100",               # experimental node-type constraint
    ),
    run_config=RunConfig(
        storage_path="s3://ckpt/pretrain-7b",  # where checkpoints are durable
        failure_config=FailureConfig(
            max_failures=10,                   # DEFAULT 0 — no retries at all.
                                               # -1 = infinite. Recovers from the
                                               # latest reported checkpoint.
            max_preemption_failures=-1,        # DEFAULT -1: preemptions retry
                                               # forever and are counted SEPARATELY
                                               # from real failures.
            controller_failure_limit=-1,       # DEFAULT -1.
        ),
        checkpoint_config=CheckpointConfig(num_to_keep=3),
    ),
)
result = trainer.fit()
```

Three things transfer directly to the Kubernetes picture. **The placement group *is* the
gang** — Ray's placement groups are atomic by construction, so `num_workers=32` either
reserves 32 bundles or waits, with no partial start; `placement_strategy` is the
topology-aware placement of module 06 expressed as a scheduling hint. **`FailureConfig`
is the failure policy**, and its default of `max_failures=0` is the same trap as an
unset JobSet `failurePolicy`: without changing it, one dead worker ends the run. **The
separation of `max_failures` from `max_preemption_failures` is the same idea as JobSet's
`…AndIgnoreMaxRestarts` actions** — do not let planned infrastructure churn consume the
budget you reserved for real faults. And the `num_workers=(min, max)` tuple is Ray's
version of `torchrun --nnodes=2:4`.

### 10. Slurm on Kubernetes, and why the split exists

Researchers with HPC muscle memory want `sbatch`, `srun`, `squeue` and array jobs; platform
teams want one declarative control plane. CoreWeave's **SUNK** resolves it by running Slurm
*as workloads on* Kubernetes: `slurmctld` and `slurmd` run in pods, nodes are shared with
Kubernetes scheduling, so researchers get the Slurm CLI while the platform keeps a single
substrate for the GPU operator, node health, and observability. CoreWeave's published
material puts the scale at >100,000 GPUs with individual jobs above 32,000 GPUs.

The transferable point is not the product, it is the observation that **a Slurm allocation
is inherently a gang** — the allocation *is* the scheduling unit, which is why HPC never
had Kubernetes' partial-placement problem in the first place (module 06 §11). Kubernetes has
spent several years absorbing that property back in, via scheduler-plugins, Volcano, Kueue,
and now the native workload APIs.

### 11. Reading a job that is not making progress

Everything above collapses into one decision procedure. Run it top to bottom.

```
  "The job isn't training." — WHICH LAYER?
  ═══════════════════════════════════════════════════════════════════════════════

  kubectl get pytorchjob ddp-demo -o jsonpath='{.status.conditions}'
        │
        ├─ no Created condition ────────────▶ webhook/CRD problem. Is the operator
        │                                     running? `kubectl get crd pytorchjobs…`
        │
  kubectl get pods -l training.kubeflow.org/job-name=ddp-demo
        │
        ├─ ZERO pods, PodGroup exists ─────▶ Volcano path: pods intentionally delayed.
        │                                     `kubectl get podgroup ddp-demo -o yaml`
        │                                     → phase Pending/Unschedulable ⇒ capacity
        │                                     or quota. This is module 06's problem.
        │
        ├─ SOME pods Pending, some Running ▶ gang scheduling is NOT working. Either
        │                                     the operator has no --gang-scheduler-name,
        │                                     or a pod's schedulerName points at the
        │                                     default scheduler. Cost bug: the Running
        │                                     pods hold GPUs and do nothing.
        │
        ├─ ALL pods Running, no step logs ─▶ RENDEZVOUS. Compare env across pods:
        │                                     kubectl exec … -- env | grep -E \
        │                                       'MASTER_ADDR|MASTER_PORT|WORLD_SIZE|PET_'
        │                                     • WORLD_SIZE differs between pods → §4
        │                                     • MASTER_ADDR unresolvable → check the
        │                                       headless Service exists
        │                                     • all agree → it is NCCL, not orchestration:
        │                                       go to 08.2 (NCCL_DEBUG=INFO)
        │
        └─ pods churning ──────────────────▶ WHICH RESTART LAYER? (§5 evidence table)
                                              restartCount up, UID same  → kubelet
                                              UID changed                → operator
                                              neither                    → torchrun agent
```

## Perspectives

**Platform-team view.** The controller's scope is narrower than its reputation: create
objects, inject env, compute a `PodGroup`, apply a restart policy. Knowing that narrow scope
is exactly what lets you attribute a stuck job to the right layer instead of guessing —
and every minute of wrong-layer guessing on a 32-GPU gang is real money.

**Researcher view.** The researcher wants `kubectl apply` or `sbatch` and nothing else. The
`TrainingRuntime`/`TrainJob` split, and SUNK, are two different answers to the same question
of who owns the complexity. A third-party treatment of the cultural split is SkyPilot's
["Slurm vs K8s for AI Infra"](https://blog.skypilot.co/slurm-vs-k8s/), which notes that even
frontier labs diverge — Slurm-flavoured HPC at Meta, Kubernetes at OpenAI's scale-out — so
this is not a vendor narrative.

**API-evolution view.** The per-framework CRD zoo unifying into `TrainJob`-over-JobSet is a
dateable migration, not a done deal: Trainer v2.3.0 shipped 2026-08-05, and its own upgrade
notes require anyone on v2.0–v2.2 to step through v2.3 before going further. `PyTorchJob`
remains what most shops actually run, and Trainer v2 still has no elastic model. Teach both,
and name the gap.

**Failure-policy view.** JobSet's rule engine is the most under-appreciated thing in this
lesson. Most operators treat failures as one undifferentiated class and set a single retry
count. JobSet lets you say "host maintenance restarts us for free, a bad node restarts one
replica, a `BackoffLimitExceeded` kills the run" — three different economic responses to
three different causes. That distinction is worth real money and is exactly what 08.8 prices.

**Economics view.** Every mechanism here maps to a term in 08.8's cost model. Gang admission
prevents burn on a partial gang. `RestartJob` instead of `RestartJobSet` shrinks the blast
radius from *N* GPUs to *N/replicas*. `InPlaceRestart` removes pod-scheduling and image-pull
from the restart critical path, cutting `R` directly. None of these are performance
optimisations; they are all cost optimisations.

## Real-world use cases

- **Kubeflow Trainer v2.3.0 (2026-08-05)** — the repository's own `CHANGELOG/CHANGELOG-2.3.md`.
  What it shows: the migration is live and has sharp edges. v2.3 moved CRDs into the Helm
  chart's template directory and **removed runtime finalizers**, both breaking changes, with
  an explicit instruction that v2.0/v2.1/v2.2 users must upgrade *through* v2.3. It also
  added a Flux Framework integration and an `OptimizationJob` CRD. Read this as evidence for
  the correct interview framing: v2 is real and moving fast, which is itself a reason most
  production clusters are still on v1.
- **JobSet `InPlaceRestart`** — `api/jobset/v1alpha2/jobset_types.go` and
  `examples/in-place-restart/`. What it shows: the industry's answer to "restart time is the
  dominant cost term at scale" is to stop recreating pods. The design detail worth carrying
  is the barrier protocol — an agent container per pod compares the pod's
  `jobset.sigs.k8s.io/in-place-restart-attempt` annotation against the JobSet's
  `status.currentInPlaceRestartAttempt` and holds the worker until every agent is in the new
  attempt. That is a distributed barrier implemented in annotations.
- **JobSet `examples/failure-policy/host-maintenance-event-model.yaml`** — a real, shipped
  example of the two-rule pattern: `RestartJobSetAndIgnoreMaxRestarts` when the child Job's
  failure reason is `PodFailurePolicy` (a maintenance event the pod failure policy
  classified), falling through to a normal `RestartJobSet` otherwise. What it shows: the
  budget-exemption idea is not theoretical; it is the reference way to run on preemptible
  capacity.
- **CoreWeave SUNK** — [blog](https://www.coreweave.com/blog/sunk-slurm-on-kubernetes-implementations),
  [docs](https://docs.coreweave.com/products/sunk), [SLUG23 deck](https://slurm.schedmd.com/SLUG23/CoreWeave-SLUG23.pdf).
  `slurmctld`/`slurmd` as pods, >100k GPUs, jobs >32k GPUs. What it shows: the target-company
  pattern for this module, with a public architecture writeup.
- **Ray Train `FailureConfig`** — `python/ray/train/v2/api/config.py`. What it shows: an
  independently designed system arriving at the same distinctions Kubernetes reached —
  separate budgets for faults and preemptions, infinite retries for the controller, recovery
  anchored on the last reported checkpoint. When two unrelated designs converge on the same
  three knobs, those knobs are the real requirements.

## Worked example

Submit the 08.5 DDP job as a `PyTorchJob` on a two-node kind cluster (CPU-only; the
rendezvous logic is identical, NCCL becomes Gloo), then break it on purpose and read each
layer.

**Step 1 — submit and confirm the object graph.**

```console
$ kubectl apply -f ddp-demo.yaml
pytorchjob.kubeflow.org/ddp-demo created

$ kubectl get pods,svc,podgroup -l training.kubeflow.org/job-name=ddp-demo
NAME                    READY   STATUS    RESTARTS   AGE
pod/ddp-demo-master-0   1/1     Running   0          22s
pod/ddp-demo-worker-0   1/1     Running   0          22s
pod/ddp-demo-worker-1   1/1     Running   0          22s
pod/ddp-demo-worker-2   1/1     Running   0          22s

NAME                        TYPE        CLUSTER-IP   PORT(S)     AGE
service/ddp-demo-master-0   ClusterIP   None         23456/TCP   22s
service/ddp-demo-worker-0   ClusterIP   None         23456/TCP   22s
service/ddp-demo-worker-1   ClusterIP   None         23456/TCP   22s
service/ddp-demo-worker-2   ClusterIP   None         23456/TCP   22s
```

Four pods, **four** headless Services — one per pod, `CLUSTER-IP None`. That is §3 step ③
made visible. (Transcript shape is representative; your ages and ordering will differ.)

**Step 2 — read the injected rendezvous env, and check the arithmetic.**

```console
$ kubectl exec ddp-demo-worker-1 -- env | grep -E 'MASTER_|WORLD_SIZE|^RANK|PET_' | sort
MASTER_ADDR=ddp-demo-master-0
MASTER_PORT=23456
PET_MASTER_ADDR=ddp-demo-master-0
PET_MASTER_PORT=23456
PET_NNODES=4
PET_NODE_RANK=2
PET_NPROC_PER_NODE=2
RANK=2
WORLD_SIZE=8
```

Read it line by line against §4. `PET_NNODES=4` is the pod count (1 master + 3 workers).
`PET_NPROC_PER_NODE=2` is what the spec asked for. `WORLD_SIZE=8` is `4 × 2` — processes,
not pods. `RANK=2` because worker-1 is worker index 1, and workers are offset by one past
the master. `MASTER_ADDR` is a *name*, resolved by the headless Service. **You wrote no
`--rdzv-endpoint` anywhere.**

Sanity-check that the name resolves from inside another pod, because a broken DNS policy is
a common cluster-level cause of "all pods Running, no progress":

```console
$ kubectl exec ddp-demo-worker-2 -- getent hosts ddp-demo-master-0
10.244.1.7      ddp-demo-master-0.research.svc.cluster.local
```

**Step 3 — read the rendezvous completing.**

```console
$ kubectl logs ddp-demo-master-0 | head -12
[INFO] torch.distributed.run: Using nproc_per_node=2.
[INFO] torch.distributed.launcher.api: Starting elastic_operator with launch configs:
  entrypoint       : train.py
  min_nodes        : 4
  max_nodes        : 4
  nproc_per_node   : 2
  rdzv_backend     : static
  rdzv_endpoint    : ddp-demo-master-0:23456
[INFO] torch.distributed.elastic.agent.server.api: [default] Rendezvous complete for
  workers. Result: restart_count=0 master_addr=ddp-demo-master-0 group_world_size=4
  group_rank=0 role_world_size=8
step 0 loss 6.912  |  step 10 loss 5.884  |  step 20 loss 5.301
```

`group_world_size=4` is agents (pods); `role_world_size=8` is processes. Both numbers
appear, which is the clearest possible confirmation of §4's distinction.

**Step 4 — break it three ways, one per layer.**

*Layer 3 (agent).* Kill one training process inside a worker pod without killing the pod:

```console
$ kubectl exec ddp-demo-worker-1 -- pkill -9 -f 'train.py'
$ kubectl get pod ddp-demo-worker-1 -o jsonpath='{.metadata.uid}{"\t"}{.status.containerStatuses[0].restartCount}{"\n"}'
0d3b6a1e-…-9f21    0
$ kubectl logs ddp-demo-worker-1 --tail=4
[WARNING] …agent.server.api: Worker group FAILED. 3/3 attempts left; will restart worker group
[INFO]    …agent.server.api: [default] Rendezvous complete… restart_count=1
step 0 loss 5.297
```

Same UID, `restartCount` still 0, `restart_count=1` in the agent log, and training resumes
from the checkpoint your script loaded. Kubernetes observed nothing.

*Layer 1 (kubelet).* With `restartPolicy: OnFailure`, kill PID 1 in the container. The pod
stays, `restartCount` increments, phase returns to Running, and nothing is emitted on the
PyTorchJob. Do it seven times with `backoffLimit: 6` and the *job* fails —
`PastBackoffLimit` summed those restarts across the whole job.

*Layer 2 (operator).* Delete a worker pod outright:

```console
$ kubectl delete pod ddp-demo-worker-2
$ kubectl get events --field-selector involvedObject.name=ddp-demo --sort-by=.lastTimestamp | tail -3
0s   Normal    SuccessfulCreatePod       pytorchjob/ddp-demo   Created pod: ddp-demo-worker-2
0s   Warning   PyTorchJobRestarting      pytorchjob/ddp-demo   job ddp-demo is restarting because Worker replica(s) failed.
```

New pod, **same ordinal**, same headless Service, same `RANK=3`. That stability is why the
name-based `MASTER_ADDR` works at all: the replacement pod re-enters the same world.

**Step 5 — cost the failure you just caused.** With `restartStrategy` semantics in mind:
this job is 4 pods × 2 procs. On a real 32×H100 gang at a blended `$3.00/GPU-hr`, a full
`RestartJobSet` that takes 6 minutes of pod scheduling + image pull + checkpoint reload
costs `32 × 3.00 × 0.1 = $9.60` per event. At Llama-3's observed rate of roughly one
interruption per 3 hours on a 16,384-GPU job — scaled down, about one per 6 days on 32 GPUs
— that is negligible. At 16,384 GPUs it is `16384 × 3.00 × 0.1 = $4,915` per event, eight
times a day. **That ratio is the entire argument for `RestartJob` and `InPlaceRestart`**, and
08.8 turns it into the `R/M` term.

## Practice — feeds the deliverable (optional stretch)

See [`../practice/survive-a-failure/README.md`](../practice/survive-a-failure/README.md)
for the full deliverable spec. **Goal:** submit a PyTorchJob on a local cluster, prove how
the rendezvous env is set, and demonstrate all three restart layers distinctly.

1. **Cluster.** `kind create cluster --config kind-2node.yaml` (two workers so placement is
   non-trivial). CPU-only is fine; set the training backend to `gloo`.
2. **Install the operator.**
   `kubectl apply -k "github.com/kubeflow/training-operator/manifests/overlays/standalone?ref=v1.9.3"`,
   then confirm `kubectl get crd pytorchjobs.kubeflow.org`. Check whether gang scheduling is
   on: `kubectl -n kubeflow get deploy training-operator -o yaml | grep gang-scheduler-name`.
   If the flag is absent, note that in your writeup — it is the §6 correction, observed.
3. **Submit** the 08.5 DDP job as a `PyTorchJob` with 1 Master + 3 Workers,
   `nprocPerNode: "2"`, `restartPolicy: ExitCode` on Worker, `backoffLimit: 3`.
4. **Capture the wiring.** `kubectl exec` into two different pods and diff their
   rendezvous env; capture `kubectl get svc`; capture the master's `Rendezvous complete`
   line showing `group_world_size` and `role_world_size`. Write one line explaining why
   `WORLD_SIZE` is `Σ replicas × nprocPerNode`.
5. **Exercise each restart layer** as in Worked example step 4, and record for each: pod
   UID before/after, `restartCount` before/after, whether a `PyTorchJobRestarting` event
   fired, and the torchrun `restart_count`.
6. **Prove the exit-code policy.** Make the training script `sys.exit(3)` once and confirm
   the job goes to `Failed` with no retry; then make it exit via `kill -9` (code 137) and
   confirm the pod is recreated.

**Acceptance:** a submitted `PyTorchJob` reaching Running with the injected rendezvous env
captured from two pods and the `WORLD_SIZE` arithmetic explained; plus a three-row table
distinguishing kubelet / operator / agent restarts by observable evidence; plus the
exit-code experiment showing 3 → job Failed and 137 → pod recreated.

## Common pitfalls

- **"The Training Operator does gang scheduling."** Half right, and the half that is wrong
  costs money. It *builds* the `PodGroup` — but only when it was started with
  `--gang-scheduler-name`, which defaults to empty, and the all-or-nothing admission itself
  is module 06's scheduler. *Symptom:* some pods Running, some Pending, job never trains.
  *Mechanism:* without a `PodGroup` (or with pods pointing at the default scheduler) each
  pod is bound greedily and independently; the bound ones hold GPUs while blocked in the
  c10d barrier.
- **"`WORLD_SIZE` is the number of pods."** It is `Σ replicas × nprocPerNode`, and it is
  silently wrong — `1` per pod — if you leave `nprocPerNode` at its `"auto"` default while a
  non-torchrun entrypoint reads the variable. *Symptom:* every rank blocks in
  `init_process_group`, GPUs at 0%, no error for the full rendezvous timeout.
- **"`backoffLimit` is per pod."** It is a whole-job budget summed over container
  `restartCount` across all Running pods of every replica type whose policy is
  `OnFailure`/`Always`/`ExitCode`. On a 64-pod job, `backoffLimit: 5` means the job dies
  after five restarts *in aggregate* — often within the first minute of a bad image.
- **"`restartPolicy: OnFailure` means the operator restarts the pod."** It means the
  *kubelet* restarts the container in place, so the pod rarely reaches `Failed` and the
  operator's own restart branch rarely runs. The two paths look identical in a summary and
  behave completely differently in an incident. Check the pod UID.
- **"`restartPolicy: Always` is the safe choice."** It retries a `sys.exit(1)` from a typo
  forever, holding the entire gang, until `backoffLimit` (if you set one) trips. Use
  `ExitCode`, which retries only signals (≥128) and fails fast on program errors.
- **"Renaming the container is cosmetic."** The v1 controller looks up the rendezvous port
  on a container literally named `pytorch` with a port named `pytorchjob-port`. Rename
  either and env generation returns "port not found"; the job creates pods with no
  `MASTER_PORT` and hangs.
- **"JobSet retries are one number."** JobSet has *four* counters and five actions
  precisely because they are not one number. Using bare `maxRestarts` with the default
  `RestartJobSet` action means a week of spot preemptions exhausts the budget you meant to
  reserve for real faults; that is what `…AndIgnoreMaxRestarts` exists to prevent.
- **"Trainer v2 is a drop-in replacement."** It renders to JobSet, has no elastic model
  yet (`TODO` in the torch plugin), and v2.3 shipped breaking changes requiring a stepped
  upgrade. Deployed reality plus direction — say both.

## Self-check

- **What exactly does the Training Operator inject so that torchrun can rendezvous, and
  what is `WORLD_SIZE`?**
  **Answer:** For each pod it creates a **headless Service** named `<job>-<rtype>-<index>`
  (`clusterIP: None`, selector on job-name/replica-type/replica-index) so every pod has a
  stable DNS name, and injects into every container: `MASTER_ADDR` = `<job>-master-0`,
  `MASTER_PORT` = the Master's `pytorchjob-port` (default 23456), `WORLD_SIZE`, `RANK`
  (master 0, worker *i* → *i*+1), `PET_NODE_RANK` (same value), `PET_NNODES` (pod count, or
  `"min:max"` in elastic mode), `PET_NPROC_PER_NODE`, `PYTHONUNBUFFERED=1`, plus the
  `PET_RDZV_*` and `PET_MAX_RESTARTS` block when `elasticPolicy` is present. **`WORLD_SIZE`
  = Σ replicas × nprocPerNode — processes, not pods** (`worldSize := totalReplicas *
  nprocPerNode`). It is only injected at all if a `Master` replica exists. Under torchrun
  these are inputs to the agent, which re-derives each process's own `RANK`/`LOCAL_RANK`.

- **A worker pod just came back. How do you tell whether the kubelet, the operator, or
  torchrun restarted it — and why does it matter?**
  **Answer:** Three observations. (a) `restartCount` incremented but pod UID and age
  unchanged → **kubelet**, container restarted in place under the pod's `restartPolicy`;
  counts toward `runPolicy.backoffLimit`. (b) Pod **UID changed** and a
  `PyTorchJobRestarting` warning event appeared on the job → **operator**, which deletes and
  recreates the pod with the same ordinal, name, Service and `RANK`. (c) Neither changed,
  but the torchrun log shows `Worker group FAILED … will restart worker group` and
  `restart_count` incremented → **the elastic agent**, invisible to Kubernetes. It matters
  because the recovery point differs: layers 1 and 2 restart the container from its
  entrypoint (so you resume from whatever checkpoint the script loads), while layer 3 forms
  a new rendezvous round with possibly a different world size — the "it restarted but from
  the wrong step" incident class.

- **You need a job that shrugs off spot preemption but fails fast on a code bug. Express
  that in PyTorchJob, and in JobSet.**
  **Answer:** In **PyTorchJob**: set `restartPolicy: ExitCode` on the Worker (and Master)
  replicas. Preemption and OOM-kill arrive as signals — SIGTERM 143, SIGKILL 137 — i.e.
  exit codes ≥ 128, which `IsRetryableExitCode` retries; a `sys.exit(1)` from a shape error
  is 1–127, which fails the job immediately with no retry. Set `backoffLimit` generously
  (it is a whole-job sum) or leave it unset. In **JobSet**: use `failurePolicy.rules` in
  order — first a rule with `action: RestartJobSetAndIgnoreMaxRestarts` matching the
  preemption signature (e.g. `onJobFailureReasons: ["PodFailurePolicy"]` with the child
  Job's own `podFailurePolicy` classifying SIGTERM), so preemptions do **not** consume the
  budget; then a rule with `action: FailJobSet` on `onJobFailureReasons:
  ["BackoffLimitExceeded"]` for genuine application failure; leaving `maxRestarts` for
  everything else. Add `restartStrategy: InPlaceRestart` if the feature gate is available,
  to keep pod scheduling out of the recovery path.

- **Where does module 06's gang scheduling plug in, and what does the operator hand it?**
  **Answer:** The operator creates exactly one `PodGroup` per job, owned by the job, in the
  job's namespace, with `minMember` = `runPolicy.schedulingPolicy.minAvailable` (defaulting
  to the sum of all replicas), `minResources` (supplied, or computed by summing `minMember`
  pods' requests), and `scheduleTimeoutSeconds`; it stamps pods with the backend's
  association marker — the label `scheduling.x-k8s.io/pod-group` for scheduler-plugins, the
  annotation `scheduling.k8s.io/group-name` for Volcano — and sets `schedulerName` if the
  pod template did not. **Module 06's scheduler then enforces all-or-nothing admission.**
  Two operational details: gang is off unless the operator was started with
  `--gang-scheduler-name`; and with Volcano the operator *withholds pod creation* until the
  `PodGroup` leaves Pending, whereas with scheduler-plugins the pods are created immediately
  and wait at `Permit` holding assumed capacity. JobSet has no `minMember` at all and
  delegates entirely to Kueue or to the native `Workload`/`PodGroup` APIs.

- **What is Trainer v2's `TrainJob` unifying, how, and what is still missing?**
  **Answer:** It replaces the per-framework CRDs (`PyTorchJob`, `TFJob`, `MPIJob`,
  `XGBoostJob`) with one `TrainJob` that references a reusable `TrainingRuntime` /
  `ClusterTrainingRuntime` — a blueprint owned by the platform team containing a **JobSet
  spec** plus an `mlPolicy` selecting a framework plugin. The controller runs a plugin
  chain: `torch` injects `PET_NNODES`/`PET_NPROC_PER_NODE`/`PET_NODE_RANK`/`PET_MASTER_ADDR`
  (= `<trainjob>-node-0-0.<trainjob>`, port 29500), `jobset` renders the JobSet,
  `coscheduling`/`volcano` build the `PodGroup` with `minMember` = Σ pod-set counts. Missing:
  **elastic world size** — the torch plugin carries an explicit TODO pending JobSet elastic
  support. Current release v2.3.0 (2026-08-05), with breaking changes that require stepping
  through v2.3 on upgrade. So: PyTorchJob is today's deployed reality, `TrainJob`-over-JobSet
  is the direction, and elasticity is the gap.

## Connections & what's next

This lesson closes the loop 08.5 opened: the CRDs and controllers here are the concrete
Kubernetes objects underneath 08.5's "drain the pod" and "resume from checkpoint" arrows,
and the `PodGroup` the operator writes is precisely the object module 06 taught the
scheduler to honour. The three-restart-layer distinction is the single most common source of
confusing incidents at this seam — know the evidence table cold for a debugging interview.
The failure-policy actions and `InPlaceRestart` are not just API trivia: they are the levers
that move the `R` (restart time) and blast-radius terms in 08.8's cost model.

Next, 08.7 leaves orchestration and failure behind for a different GPU-money problem — an
under-fed input pipeline starving an otherwise perfectly healthy, perfectly scheduled,
perfectly rendezvoused job — using the same SM%/cost lens this module has used since 08.3.

## References & further reading

**Primary sources**

1. **`kubeflow/training-operator` at tag `v1.9.3`** — <https://github.com/kubeflow/training-operator>.
   **Read `pkg/apis/kubeflow.org/v1/pytorch_types.go`, `common_types.go`,
   `pytorch_defaults.go`, `pkg/controller.v1/pytorch/{envvar,master,elastic}.go`,
   `pkg/controller.v1/common/{job,pod,service,scheduling}.go`, `pkg/core/{job,pod}.go`.**
   Source of every default and every mechanism in §2–§6: `WORLD_SIZE = totalReplicas ×
   nprocPerNode`, `nprocPerNode` default `"auto"` and `getNprocPerNodeInt()` returning 1 for
   non-integers, port name `pytorchjob-port` / 23456, `restartPolicy` default `OnFailure`,
   `cleanPodPolicy` default `None`, `IsRetryableExitCode(code) = code >= 128`,
   `PastBackoffLimit` summing container restart counts, per-pod headless Services with
   `ClusterIP: None`, and `SyncPodGroup`'s `minMember` default. Verified by fetching the tag
   in this pass. **Correction recorded:** an earlier version of this lesson stated
   `WORLD_SIZE` is the pod count and that the operator creates a single headless Service for
   the master; both are wrong against this source.
2. **`kubeflow/trainer` (the same repository at `master`), VERSION `v2.3.0`** —
   <https://github.com/kubeflow/trainer>. **Read `pkg/apis/trainer/v1alpha1/trainjob_types.go`,
   `pkg/runtime/framework/plugins/torch/torch.go`,
   `pkg/runtime/framework/plugins/coscheduling/coscheduling.go`, `pkg/constants/constants.go`,
   `manifests/base/runtimes/torch_distributed.yaml`, `CHANGELOG/CHANGELOG-2.3.md`.** Source of
   the `TrainJob`/`TrainingRuntime` shape, the `PET_*` injection with `PET_MASTER_ADDR =
   <trainjob>-node-0-0.<trainjob>` and port 29500, the coscheduling `minMember` = Σ pod-set
   counts, the elastic TODO, and the v2.3.0 release date of 2026-08-05 with its breaking
   changes. Verified by cloning at HEAD.
3. **`kubernetes-sigs/jobset` at `master`** — <https://github.com/kubernetes-sigs/jobset>.
   **Read `api/jobset/v1alpha2/jobset_types.go` in full; it is the single best-documented
   API in this lesson.** Source of the five `FailurePolicyAction` values and their exact
   counter semantics, the three `JobSetRestartStrategy` values including the `InPlaceRestart`
   barrier protocol, `Network{enableDNSHostnames, subdomain, publishNotReadyAddresses}`,
   `Coordinator`, `DependsOn`, the `alpha.jobset.sigs.k8s.io/exclusive-topology` annotation,
   and the label/annotation vocabulary. Also read `examples/failure-policy/` and
   `site/content/en/docs/workload-aware-scheduling/gang_scheduling.md` with its
   `Workload`/`PodGroup`/`Gang{minCount: 6}` example. Verified by cloning at HEAD.
4. **`ray-project/ray`, `python/ray/train/v2/api/config.py` and `python/ray/air/config.py`** —
   <https://github.com/ray-project/ray>. Source of `ScalingConfig(num_workers, use_gpu,
   resources_per_worker, placement_strategy default "PACK", accelerator_type)`, the elastic
   `(min, max)` tuple form, and `FailureConfig(max_failures` default **0**`,
   controller_failure_limit` default **−1**`, max_preemption_failures` default **−1**`)`.
   Fetched directly from the repository; docs.ray.io is unreachable from this environment.
5. **PyTorch elastic / `torchrun`** — the `PET_*` environment contract in this lesson is
   read off the operator's generators (sources 1 and 2), each of which maps one-to-one onto a
   `torchrun` flag (`--nnodes`, `--nproc-per-node`, `--node-rank`, `--rdzv-backend`,
   `--rdzv-endpoint`, `--rdzv-id`, `--rdzv-conf`, `--max-restarts`, `--standalone`).
   *docs.pytorch.org is blocked by this environment's egress proxy; the flag semantics were
   not re-read from the upstream tutorial in this pass and are stated only where the operator
   source names them.*

6. **Kubeflow Trainer docs** — <https://www.kubeflow.org/docs/components/trainer/>, including
   the [legacy-v1 PyTorch user guide](https://www.kubeflow.org/docs/components/trainer/legacy-v1/user-guides/pytorch/)
   and the [v2 migration guide](https://www.kubeflow.org/docs/components/trainer/operator-guides/migration/).
   *kubeflow.org is unreachable from this environment; listed as the human-readable
   companion to source 2, not as the basis for any claim here.*
7. **JobSet documentation site** — <https://jobset.sigs.k8s.io/docs/>. *Unreachable from this
   environment; the equivalent content was read from the `site/content/en/docs` tree inside
   the repository (source 3).*
8. **Kubeflow blog — Trainer v2 GA and release notes** — <https://blog.kubeflow.org/trainer/intro/>
   and the per-release posts. *Unreachable from this environment; the version and date claims
   in this lesson come from the repository's own `VERSION` file and `CHANGELOG/` directory
   instead.*

**Real-world engineering blogs**

9. **CoreWeave — "A Slurm on Kubernetes Implementation for HPC and Large Scale AI"** —
   <https://www.coreweave.com/blog/sunk-slurm-on-kubernetes-implementations>, with
   [SUNK docs](https://docs.coreweave.com/products/sunk) and the
   [SLUG23 deck](https://slurm.schedmd.com/SLUG23/CoreWeave-SLUG23.pdf). The >100k-GPU /
   >32k-GPU-per-job figures are CoreWeave's own published claims. *These hosts are blocked
   from this environment; treat the numbers as vendor-published and dated.*
10. **PyTorch blog — "PyTorch on Kubernetes: Kubeflow Trainer Joins the PyTorch Ecosystem"** —
    <https://pytorch.org/blog/pytorch-on-kubernetes-kubeflow-trainer-joins-the-pytorch-ecosystem/>.
    First-party evidence that Trainer v2 is the endorsed direction rather than a
    Kubeflow-only claim. *pytorch.org is blocked here; not fetched in this pass.*
11. **SkyPilot — "Slurm vs K8s for AI Infra"** — <https://blog.skypilot.co/slurm-vs-k8s/>.
    A third-party treatment of the Slurm/Kubernetes cultural split. *Not fetched in this
    pass; cited as a perspective, not as a source of fact.*

**Related lessons in this course** (internal links, not external sources — not counted in
`sources:`)

- **[06.2 · Gang scheduling: all-or-nothing admission](../../06-scheduling-capacity/lessons/02-gang-scheduling.md)** —
  the `PodGroup` CRD field by field, the five extension points, `permitWaitingTimeSeconds`
  / `podGroupBackoffSeconds` / `podGroupRejectPercentage` defaults, Kueue's Workload-level
  alternative, and the native `Gang{minCount}` successor. **This lesson deliberately does
  not re-teach any of it.**
- **[08.5 · Failure & elasticity](05-failure-and-elasticity.md)** — the detection→drain→
  re-rendezvous→resume loop that layer 3 of §5 implements.
- **[08.8 · Training economics](08-training-economics.md)** — where the restart-time and
  blast-radius consequences of every failure-policy choice here become the `R/M` term in a
  dollar figure.

> **Snapshot (2026-08).** Version-specific claims — Training Operator `v1.9.3`, Kubeflow
> Trainer `v2.3.0` (2026-08-05), JobSet `v1alpha2` at `master`, Ray Train v2 API at `master`
> — are dated against fast-moving projects. Re-read the type files before quoting a default
> in a design review.
