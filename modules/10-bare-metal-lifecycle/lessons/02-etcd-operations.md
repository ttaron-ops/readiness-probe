---
lesson: "10.2"
title: "etcd operations: disk, quorum, and the 2am restore"
module: "10"
concept: "etcd operations: disk, quorum, and the 2am restore"
status: not-started
est_time: "12h"
prev: "01-cluster-provisioning.md"
next: "03-control-plane-ha.md"
artifacts: []
sources: 8
---

# 10.2 · etcd operations: disk, quorum, and the 2am restore

> **Concept.** You know what etcd is — now you own its disk, its quorum, its NOSPACE alarm, and its disaster-recovery runbook.
>
> Module: [🖥️ 10 — Bare metal and cluster lifecycle](../README.md) · Deliverable: [Capex-vs-cloud + KTHW/etcd writeup](../practice/capex-vs-cloud/README.md)

## Where this fits

Lesson 10.1 got you a working control plane and the PKI to prove it: you can now name every cert
between the apiserver and etcd and trace a write end to end. This lesson is the **anchor of the
whole module** — per the module README, it's "the thing that pages you" — because everything else
in this module (HA topology in 10.3, declarative fleets in 10.4, even the health-remediation loop
in 10.6) assumes the datastore underneath the control plane stays up. 10.1 built the pipe between
apiserver and etcd; this lesson is everything that can go wrong on the etcd side of that pipe once
real write traffic hits it — quota exhaustion, disk latency, quorum loss, and how you get the
cluster writable again under time pressure.

## Why this matters

etcd is the single stateful thing in the cluster and the thing that will actually page you. On EKS/GKE, AWS/Google owned etcd's disk, its backups, and its quorum — you never got that page. On bare metal you do. A full etcd turns the **entire cluster read-only** (no deploys, no scale-ups, no GPU-job scheduling); a lost quorum stops the API cold; a corrupt data dir means restoring from a snapshot under time pressure. This is not hypothetical: a documented kOps incident (below) shows a real team hitting a NOSPACE alarm in production and working the exact compact/defrag/disarm drill this lesson teaches, live, under an active outage. This lesson is the runbook you will page yourself with at 2am — the module's checkpoint gates on recovering etcd from a quota-exceeded read-only state **in under 30 minutes**, timed, unaided.

## What's new here (calibration)

Per this module's calibration (see the [README](../README.md#calibrated-to-your-background---what-we-skip)):
Module 02 taught etcd's **role**: the single source of truth, the only thing the apiserver
persists to, watch/revision semantics behind informers. It answered *what etcd is for*. You also
already have 04's GPU Operator experience, 05's XID/NPD concepts, and general on-prem fluency —
none of that is re-taught here either.

This lesson is **operations** — nothing here was in 02:
- **Quorum math** and how many failures 3 vs 5 members survive.
- The **`--quota-backend-bytes`** space quota, the **NOSPACE alarm**, and the read-only-cluster failure mode it triggers.
- **Compaction vs defragmentation** — two different reclamations people constantly conflate.
- **Snapshot save / restore**, including the etcd **3.6 tooling split** (save stays in `etcdctl`; **restore and offline defrag moved to `etcdutl`**).
- **Why etcd dies on slow disks** — WAL `fsync` latency — and the exact metric that warns you first.

## Core concepts

### Quorum: why odd numbers, and failure tolerance
etcd uses Raft; a write commits only after a **majority (quorum = ⌊N/2⌋+1)** of members persist it. Tolerated simultaneous failures = `N − quorum`.

| Members N | Quorum | Failures tolerated |
|-----------|--------|--------------------|
| 1 | 1 | 0 |
| 3 | 2 | **1** |
| 5 | 3 | **2** |
| 7 | 4 | 3 |

This is self-check (a): 3 tolerates **1** failure, 5 tolerates **2** — because quorum is 2 and 3 respectively, and you can lose N−quorum and still form a majority. **Even sizes are strictly worse**: 4 members still need 3 for quorum, so it tolerates only 1 failure — same as 3 but with more machines to fail. That's why you run **3, 5, or 7**, never 4 or 6. Lose quorum and etcd goes read-only and elects no leader until a majority returns; recovering a permanently-lost-majority cluster means `--force-new-cluster` from a surviving member or a snapshot restore. 5 is the sweet spot for a large fleet (survive two failures, e.g. one dead node during a rolling upgrade); 7 buys little and slows every write (more members to fsync).

### The space quota, the NOSPACE alarm, and the read-only cluster
etcd keeps every write in a backend (bbolt/mmap file). To stop unbounded growth it enforces **`--quota-backend-bytes`**: default **2 GiB** if unset/zero. etcd's own documentation recommends staying at or under **8 GiB** (`8*1024*1024*1024`) and calls larger sizes "untested" — this is a **documented recommendation tied to MTTR, not a hard technical wall**. The reasoning is concrete: on typical hardware today, a ~2 GB restore takes roughly 20 seconds; etcd's own guidance is that materially larger data sizes don't restore in a time you'd want during an incident. You *can* raise the quota on a high-memory host with fast NVMe, but you are trading a longer worst-case recovery time for it — quantify that tradeoff explicitly rather than treating 8 GiB as an immovable ceiling or, worse, forgetting it's adjustable at all.

When the backend DB size crosses the quota, etcd raises a cluster-wide **`NOSPACE` alarm** and **refuses all writes**, permitting only reads and the maintenance calls needed to recover.

Downstream, the apiserver's writes fail (`etcdserver: mvcc: database space exceeded`), so **the whole Kubernetes cluster goes read-only**: `kubectl apply` fails, controllers can't update status, the scheduler can't bind, nothing new lands. Reads still work, which is why it's insidious — dashboards look alive.

Recovery sequence (drill 1):
```bash
# 1. Confirm the alarm
etcdctl endpoint status --cluster -w table
etcdctl alarm list                       # NOSPACE

# 2. Compact history up to the current revision (frees logical space)
rev=$(etcdctl endpoint status -w json | grep -o '"revision":[0-9]*' | head -1 | cut -d: -f2)
etcdctl compact "$rev"

# 3. Defragment to actually return the freed pages to the filesystem
etcdctl defrag --cluster              # or per-endpoint; do members one at a time in prod

# 4. Disarm the alarm so writes resume
etcdctl alarm disarm
```
Order matters: **compact → defrag → disarm**. Disarming before you've actually reclaimed space just re-trips the alarm on the next write. Do `defrag` **one member at a time** — defrag blocks that member (it's a stop-the-world rewrite of the DB file), and hitting all members at once can drop you below quorum. This is exactly the sequence a real kOps team ran under a live incident — see [Real-world use cases](#real-world-use-cases).

### Alarm types
etcd has two alarms, both cluster-wide and both blocking writes until disarmed:
- **`NOSPACE`** — backend exceeded `--quota-backend-bytes` (the drill above).
- **`CORRUPT`** — a member detected a data hash mismatch (raised by periodic corruption checking, `--experimental-corrupt-check-time`). CORRUPT is not something you `defrag` away; it means a member's data diverged and you likely restore that member from a snapshot or re-add it fresh.
`etcdctl alarm list` shows which; never blindly `alarm disarm` a CORRUPT without understanding why it fired.

### Compaction vs defragmentation (self-check b)
Different reclamations, both needed:
- **Compaction** discards **old key revisions/history** below a given revision. etcd is MVCC — every update keeps the prior version so watches can replay. Compaction drops that superseded history. It reduces the **logical keyspace** but does **not shrink the on-disk file**; the freed pages become internal free space in bbolt. Kubernetes auto-compacts every 5 min via the apiserver's `--etcd-compaction-interval` (default `5m0s`).
- **Defragmentation** rewrites the backend file to **release those free pages back to the filesystem**, shrinking the actual DB file (`db size` vs `db size in use` in `endpoint status`). It's disruptive (blocks the member) and is **not** automatic — you schedule it.

Mnemonic: **compaction frees history inside the file; defrag returns space to the disk.** You compact often (cheap, automatic); you defrag occasionally and carefully (blocking, manual, one member at a time).

Two compaction knobs, don't confuse them: the **apiserver** drives compaction on etcd via `--etcd-compaction-interval` (default `5m`), and etcd *itself* has `--auto-compaction-retention` / `--auto-compaction-mode` (periodic or revision-based) as a backstop. In a kubeadm cluster the apiserver is the active compactor; you mostly leave etcd's own retention at the default and make sure exactly one thing is compacting so history isn't kept longer than intended (which would grow the DB toward the quota).

### Why etcd hates slow disks (self-check c)
Every Raft commit must be **durably persisted before it's acknowledged**: etcd appends the entry to its **WAL and calls `fsync`**, and the write isn't committed until fsync returns on a quorum. So the dominant latency is **disk `fsync` (write-ahead-log sync) latency**, not CPU or network. On slow/contended disks (spinning rust, oversubscribed cloud EBS, a noisy-neighbor NVMe) fsync stalls → commits stall → the leader misses heartbeats → **leader elections churn**, and the whole API latency balloons.

The metric that warns you first: **`etcd_disk_wal_fsync_duration_seconds`** (histogram; watch the p99). The companion is **`etcd_disk_backend_commit_duration_seconds`** (bbolt commit). Rule of thumb from the etcd docs: WAL fsync p99 should be **well under ~10ms**; sustained tens-of-ms means the disk can't keep up. Also watch `etcd_server_leader_changes_seen_total` (should be ~flat) and heartbeat-send failures. Fix: **dedicated fast NVMe/SSD for the etcd data dir**, never shared with the container runtime or logs, and raise `--heartbeat-interval`/`--election-timeout` only as a last resort. On a GPU fleet this is real money-versus-reliability: etcd disk is cheap insurance against a fleet-wide stall — a few hundred dollars of NVMe versus an hour of idle H100s is not a close call.

### Snapshots and restore — the 3.6 tooling split
Back up with a point-in-time snapshot:
```bash
ETCDCTL_API=3 etcdctl snapshot save /backup/etcd-$(date +%F-%H%M).db \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key
```
**etcd 3.6 change (Kubernetes 1.33+):** `snapshot save` **stays in `etcdctl`**. Everything offline moved to **`etcdutl`**, the offline/data-file tool:
- **`snapshot restore`** — removed from `etcdctl`, now `etcdutl snapshot restore`.
- **`snapshot status`** — likewise moved: `etcdutl snapshot status <file>` (not `etcdctl`).
- **Offline `defrag`** — this is the one people miss. `etcdctl defrag` is now **online-cluster-only** (it defrags a *running* member over its client endpoint, e.g. `etcdctl defrag --cluster`). To defragment a **stopped** member's data directory directly on disk — the offline case, e.g. as part of a DR procedure before you've restarted etcd — the command is now **`etcdutl defrag --data-dir <dir>`**, not `etcdctl`. Confirmed directly in the etcd project's own changelog (PR #13809, "Removed etcdctl snapshot restore," which moved snapshot restore and snapshot status to `etcdutl` alongside this defrag split — see [References](#references--further-reading)).

On 3.5 and earlier, restore was `etcdctl snapshot restore` and offline defrag ran through `etcdctl` too; hard-coding either in a 3.6 runbook fails with a command-not-found or an error telling you to use `etcdutl`. If your runbook was written before 3.6, audit it for both `snapshot restore` and any offline `defrag` step.

Restore (offline — etcd must be **stopped**), etcd 3.6+:
```bash
etcdutl snapshot restore /backup/etcd-....db \
  --name m1 \
  --initial-cluster m1=https://10.0.0.1:2380 \
  --initial-advertise-peer-urls https://10.0.0.1:2380 \
  --data-dir /var/lib/etcd-restored
```
Then point etcd's `--data-dir` at the restored dir and start it. Restore **initializes a new logical cluster** (new member IDs) — for a multi-member cluster you restore on each member with matching `--initial-cluster`, then start them together. On a kubeadm stacked control plane: stop the static pod (`mv /etc/kubernetes/manifests/etcd.yaml /tmp`), restore, swap the data dir, move the manifest back.

**Multi-member restore** is the sharp edge: you do **not** restore on one member and let it replicate. You restore the *same snapshot on every member*, each with its own `--name` and its own peer URL but an **identical `--initial-cluster`** list, then start them together — they form a fresh Raft cluster with new member IDs that agree on the snapshot's data. Restoring on one and joining the others as blank members will not reconstruct the data. Because restore mints a new cluster identity, always take members that survived a partial failure *out* before restoring; don't mix restored and live members.

### Backups are a policy, not a command
A snapshot you took once and never verified is not a backup. On a real fleet:
- Cron `snapshot save` on a leader endpoint every N minutes; ship the `.db` **off the node** (object storage), because a node-local backup dies with the node.
- **Verify** each snapshot (`etcdutl snapshot status` on 3.6+ — check the hash and revision) and periodically **rehearse the restore** on a throwaway VM. An untested restore runbook is the one that fails at 2am. The CNCF "Surgeon's Handbook" walkthrough (below) is a good template for a narrower, surgical restore that doesn't require a full-cluster rebuild — worth having as an alternative to the full DR runbook when you only need to recover part of the keyspace.
- Know your **RPO**: the window between snapshots is data you lose on a full-cluster restore (drill 2 shows exactly which objects vanish). For a busy scheduler, 5-minute snapshots mean up to 5 minutes of lost pod/lease state.

### Sizing and the monitoring you actually alert on
etcd is small and latency-bound, not throughput-bound. A large cluster's etcd is happy on a few CPUs, 8–16 GB RAM, and — the part that matters — a **dedicated low-latency SSD/NVMe** for the data dir, isolated from logs and the container runtime. Keep the DB comfortably under the ~8 GiB documented-recommendation ceiling; if you're approaching it, something is leaking (unbounded Events, a controller hot-looping writes, huge ConfigMaps) rather than "we need to raise the quota." The alerts worth paging on:

| Signal | Metric | Rough threshold |
|--------|--------|-----------------|
| Disk too slow | `etcd_disk_wal_fsync_duration_seconds` p99 | sustained > ~10ms |
| Backend commit slow | `etcd_disk_backend_commit_duration_seconds` p99 | sustained > ~25ms |
| Election churn | `etcd_server_leader_changes_seen_total` | any sustained rate |
| No leader | `etcd_server_has_leader` | == 0 (page immediately) |
| Approaching quota | `etcd_mvcc_db_total_size_in_bytes` | > ~70% of quota |
| Proposal failures | `etcd_server_proposals_failed_total` | rising |

`etcd_server_has_leader == 0` means quorum is lost — that's the "cluster API is frozen" page. `wal_fsync` p99 climbing is the *early* warning that precedes it, which is why it's the answer to "what warns you first." A CNCF blog on `etcd-diagnosis` tooling (below) is a good further-reading pointer for triaging exactly this class of production incident faster than reading raw metrics by hand.

### Defrag cadence — automate it, carefully
Defrag is manual by design, but "manual" shouldn't mean "never." Common practice: a periodic job that defrags **one member at a time, only if `db size` − `db size in use` exceeds a threshold**, waiting for each member to report healthy before moving to the next, and skipping the leader (or stepping leadership off it first). Never parallel-defrag — each defrag blocks its member, and blocking two of three members at once loses quorum. `etcdctl endpoint status --cluster -w table` shows `DB SIZE` per member so you can see fragmentation before deciding to run.

### Diagnosing the read-only cluster at 2am
Symptom is generic: `kubectl apply` hangs or errors, controllers stall. Three distinct root causes, distinguished fast:
1. **NOSPACE** — `etcdctl alarm list` shows it; writes error `mvcc: database space exceeded`. Fix = compact/defrag/disarm.
2. **Lost quorum** — `etcd_server_has_leader == 0`, `endpoint status` shows members unreachable; apiserver logs `context deadline exceeded` to etcd. Fix = restore members / `--force-new-cluster` from a survivor.
3. **Slow disk** — no alarm, has a leader, but `wal_fsync` p99 is huge and everything is molasses. Fix = move etcd to faster disk / remove the noisy neighbor.
Check `alarm list` → `has_leader` → `wal_fsync` in that order and you've triaged it in under a minute.

### Learner members and safe growth
When you add a member to a large cluster, add it as a **learner** first (`etcdctl member add --learner`). A learner receives the Raft log but **doesn't count toward quorum** and can't vote, so a slow-to-catch-up new member can't destabilize quorum or trigger an election while it snapshots the existing data. Promote it (`member promote`) once it's caught up. Growing 3→5 naively (two voting members that haven't synced) is a classic way to wedge a busy cluster.

### The DR runbook (single-cluster data loss)
The ordered procedure to keep next to the pager, for a stacked kubeadm control plane where etcd data is gone/corrupt:
1. **Stop the bleeding.** Stop each apiserver static pod (`mv kube-apiserver.yaml /tmp`) so nothing writes half-state during recovery.
2. **Stop etcd** on every member (`mv etcd.yaml /tmp`). Restore is offline.
3. **Pick the snapshot.** Newest verified `.db` from off-node storage (`etcdutl snapshot status` to confirm hash/revision). Note the revision — everything after it is lost (your RPO).
4. **Restore on each member** with `etcdutl snapshot restore`, identical `--initial-cluster`, per-member `--name`/`--initial-advertise-peer-urls`, into a fresh `--data-dir`.
5. **Point etcd at the restored data dir**, move `etcd.yaml` back on all members, let them form quorum. Verify `etcd_server_has_leader == 1` and `endpoint status --cluster`.
6. **Bring apiservers back** (`mv kube-apiserver.yaml` into place). Confirm `kubectl get nodes` and that controllers resume.
7. **Reconcile drift.** Objects created after the snapshot are gone; re-apply from GitOps/manifests. Rotate any Secrets that may have been exposed if the loss involved a stolen data dir.
The discipline that makes this work is set *before* the incident: off-node verified snapshots (RPO you've chosen) and a restore you've rehearsed on a throwaway VM.

## Perspectives

**Developer/app-owner view.** From inside a Deployment, a NOSPACE-frozen cluster looks like `kubectl apply` silently hanging or timing out — nothing tells the app owner "etcd is full." They escalate a "deploy is stuck" ticket with no idea the real cause is two hops away in the datastore. Part of running this well is making the NOSPACE/quorum state visible before it reaches that ticket.

**Operator/on-call view.** This is the lesson written for the person holding the pager. The triage order (`alarm list` → `has_leader` → `wal_fsync`) exists because under time pressure you don't get to read every metric — you need one ordered checklist that discriminates the three failure classes in under a minute, which is exactly the shape of the module's timed checkpoint.

**Hardware/disk view.** Everything in this lesson traces back to one physical fact: fsync on a spinning or contended disk takes milliseconds to tens-of-milliseconds, and Raft can't acknowledge a write until it's durable on a quorum. etcd's operational character — its quota, its compaction/defrag split, its election-churn failure mode — is downstream of that single physical constraint. Buying a dedicated NVMe for the data dir isn't a nice-to-have; it's the fix for the root cause of most etcd pages.

**Economics view.** The 8 GiB documented-recommendation quota is explicitly an MTTR tradeoff: a 2 GB restore takes ~20 seconds on typical hardware, and etcd's own guidance is that materially larger sizes haven't been validated to restore acceptably fast. On a GPU fleet, minutes of control-plane downtime translate directly into idle, still-billed-for GPU-hours — so the "cheap" move of raising the quota to avoid ops work can be the expensive move if it stretches your worst-case restore time. Frame this explicitly in the capex-vs-cloud writeup: etcd disk and quota discipline are cheap insurance against very expensive idle compute.

## Real-world use cases

- **"Kubernetes ETCD Out of Space on kOps: A Real-Life Incident and Recovery"** — <https://medium.com/@caue._/kubernetes-etcd-out-of-space-on-kops-a-real-life-incident-and-recovery-a1857f3e0998> — a documented, real NOSPACE-alarm incident and recovery on a kOps cluster. Shows the exact "quota exceeded → read-only cluster → compact/defrag/disarm" drill as it actually played out for a real team under an active incident, not as a lab exercise. This is the primary use-case reference for Practice Drill 1 below.
- **CNCF "The Kubernetes Surgeon's Handbook: Precision Recovery from etcd Snapshots"** — <https://www.cncf.io/blog/2025/05/08/the-kubernetes-surgeons-handbook-precision-recovery-from-etcd-snapshots/> — a walkthrough of surgical recovery from etcd snapshots without a full-cluster restore. A useful companion to Drill 2: shows a narrower-scope alternative to the full DR runbook above, for when you only need to recover part of the keyspace.
- **CNCF "Making etcd incidents easier to debug in production Kubernetes"** — <https://www.cncf.io/blog/2026/03/12/making-etcd-incidents-easier-to-debug-in-production-kubernetes/> — introduces `etcd-diagnosis` tooling built specifically for triaging real production etcd incidents. Directly relevant to the "diagnosing the read-only cluster at 2am" section above — a pointer to tooling beyond raw `etcdctl`/metrics triage.

## Worked example — trace of the two drills on a stacked kubeadm node

**Drill 1 (quota → read-only → recover):** Push junk into etcd until `db size` exceeds a tiny quota you set (`--quota-backend-bytes=$((16*1024*1024))` for testing), e.g. loop `etcdctl put junk/$i <64KB>`. Watch `etcdctl alarm list` flip to `NOSPACE`; simultaneously `kubectl create deploy x --image=nginx` fails with `mvcc: database space exceeded`. Recover: get current revision → `etcdctl compact <rev>` → `etcdctl defrag` → `etcdctl alarm disarm`. Confirm `kubectl apply` succeeds again and `endpoint status` shows `db size` shrunk. Document what the apiserver returned before and after. This mirrors the real kOps incident above almost step for step.

**Drill 2 (destroy data dir → restore):** `etcdctl snapshot save` a good backup. Stop the etcd static pod (`mv /etc/kubernetes/manifests/etcd.yaml /tmp`), `rm -rf /var/lib/etcd/member` (the data dir). The apiserver now crashloops (no etcd). Restore: `etcdutl snapshot restore <snap> --data-dir /var/lib/etcd --name <node> --initial-cluster <node>=https://<ip>:2380 --initial-advertise-peer-urls https://<ip>:2380`, then `mv /tmp/etcd.yaml /etc/kubernetes/manifests/`. The kubelet restarts etcd, the apiserver recovers, and `kubectl get` shows exactly the cluster state as of the snapshot revision — anything created after the snapshot is gone. Note the revision gap in your writeup.

## Practice
On a 1- or 3-VM cluster (KVM/multipass/small cloud VMs), stacked kubeadm etcd is fine.

**Drill 1 — fill and recover:** set a tiny `--quota-backend-bytes`, fill past it, observe the NOSPACE alarm and the apiserver going read-only (capture the exact write error and a failing `kubectl apply`), then recover via `compact → defrag → alarm disarm`. Record `endpoint status` `db size` before/after. Compare your own timing against the module checkpoint's < 30-minute target.

**Drill 2 — destroy and restore:** `etcdctl snapshot save`, verify with `etcdutl snapshot status`, create a marker object (`kubectl create cm dr-marker --from-literal=t=$(date +%s)`) *after* the snapshot so you can prove it disappears, destroy the data dir, watch the apiserver crashloop (`crictl logs` on the apiserver container / `journalctl -u kubelet`), restore with **`etcdutl snapshot restore`** (confirm you're on 3.6+ via `etcd --version` / `kubectl version` ≥1.33 — the 3.5 `etcdctl snapshot restore` command is gone), and prove recovery: apiserver healthy, `kubectl get nodes` Ready, and the `dr-marker` ConfigMap **absent** (it was created after the snapshot revision).

**Drill 3 (optional, tooling-split practice) — offline defrag on 3.6:** stop a member, run `etcdutl defrag --data-dir <dir>` directly against its stopped data directory, and confirm the file shrinks — then try the same thing with `etcdctl defrag` against the stopped member and confirm it fails (etcdctl's defrag is online-only on 3.6+). This is the fastest way to internalize the tooling split rather than just reading about it.

**Acceptance:** both failure drills documented with **exact commands, the etcd tool used (and the 3.5-vs-3.6 restore and offline-defrag difference), and what the apiserver did** at each step — feeds the deliverable's etcd writeup alongside the KTHW cert table from 10.1.

## Common pitfalls

- **Treating the 8 GiB quota as an immovable technical limit.** It's a documented recommendation tied to restore-time MTTR (etcd's own docs call larger sizes "untested"), not a hard ceiling — you can raise it on a high-memory, fast-disk host, but you're explicitly trading away restore speed to do it. Decide that tradeoff on purpose, don't inherit it by accident.
- **Disarming the NOSPACE alarm before compacting and defragging.** `alarm disarm` alone doesn't free any space; the next write re-trips it immediately. The order is always compact → defrag → disarm.
- **Defragging all members in parallel.** Each `defrag` blocks that member entirely (it's a stop-the-world file rewrite). Defragging two of three members at once can drop the cluster below quorum mid-recovery — always one member at a time, waiting for health before moving to the next.
- **Restoring a snapshot on one member and expecting the others to catch up by replication.** Restore mints a brand-new logical cluster with new member IDs; you must restore the *identical* snapshot on every member with a matching `--initial-cluster`, then start them together. A "restore one, rejoin the rest" plan will not reconstruct the data.
- **Hard-coding etcd 3.5-era commands in a 3.6+ runbook.** `etcdctl snapshot restore` and `etcdctl snapshot status` are gone on 3.6 (moved to `etcdutl`), and `etcdctl defrag` is now online-cluster-only — offline defrag against a stopped member's data dir is `etcdutl defrag --data-dir <dir>`. A runbook written against 3.5 fails or behaves differently on 3.6 without anyone noticing until an actual incident.

## Self-check
**(a) 3-member vs 5-member etcd — how many simultaneous member failures does each tolerate, and why?**
**Answer:** 3 tolerates **1**, 5 tolerates **2**. A write needs a majority (quorum = ⌊N/2⌋+1 = 2 and 3); you can lose `N − quorum` members and still form a majority. Even sizes don't help (4 also needs 3, tolerates only 1), which is why you run 3/5/7.

**(b) Compaction vs defragmentation — what does each reclaim?**
**Answer:** **Compaction** discards superseded MVCC key **history** below a revision — frees logical space inside the DB file but doesn't shrink it (auto every 5 min via the apiserver). **Defragmentation** rewrites the backend file to **return freed pages to the filesystem**, actually shrinking `db size` — disruptive, manual, one member at a time.

**(c) Why does etcd hate slow disks — what latency dominates, what metric warns you?**
**Answer:** Every Raft commit must `fsync` the WAL to disk before it's acknowledged, so **WAL fsync latency dominates** — slow disks stall commits, miss heartbeats, and churn leader elections. First warning metric: **`etcd_disk_wal_fsync_duration_seconds`** (watch p99; keep it well under ~10ms), backed up by `etcd_disk_backend_commit_duration_seconds` and `etcd_server_leader_changes_seen_total`.

**(d) What changed in etcd 3.6's tooling around snapshot restore and offline defrag, and why does it matter for your runbook?**
**Answer:** `snapshot save` stays in `etcdctl`, but **`snapshot restore` and `snapshot status` moved to `etcdutl`**, the offline/data-file tool. Separately, **`etcdctl defrag` became online-cluster-only**; defragmenting a *stopped* member's data directory directly is now `etcdutl defrag --data-dir <dir>`. It matters because a runbook written for 3.5 (`etcdctl snapshot restore`, `etcdctl defrag` on a stopped member) breaks on 3.6+ clusters (Kubernetes 1.33+) — exactly the kind of stale-runbook gap that turns a routine restore into a longer outage.

**(e) You're paged with a frozen `kubectl apply`. What three checks triage the root cause in under a minute, and what fixes each?**
**Answer:** Check in order: (1) `etcdctl alarm list` — if `NOSPACE`, fix is compact → defrag → disarm. (2) `etcd_server_has_leader` — if `0`, quorum is lost; fix is restoring members or `--force-new-cluster` from a survivor. (3) `etcd_disk_wal_fsync_duration_seconds` p99 — if elevated with a leader present and no alarm, the disk is too slow; fix is moving etcd to faster, dedicated storage. This order distinguishes the three most common root causes fastest.

## Connections & what's next

This lesson assumed the PKI from **10.1** already works — the apiserver reaching etcd at all depends on `apiserver-etcd-client` being signed by the right CA, which is why 10.1 comes first. It also feeds directly into **10.3 (control-plane HA)**: the quorum math here (why 3 tolerates 1 failure, why stacked etcd ties an apiserver node's death to an etcd member's death) is the exact reasoning 10.3 builds its HA topology decision on, and the DR runbook here is what you rehearse *before* you trust a 3-node HA cluster in production. It also underpins **10.6 (hardware health, remediation & RMA)**: a node RMA'd out from under a stacked-etcd member is a quorum event, not just a compute event — you'll want this lesson's quorum math in your head when you design the drain/cordon/RMA loop. And the 8 GiB-quota-vs-MTTR tradeoff here is a direct input to the **10.8 capex-vs-cloud** model: etcd disk quality is cheap insurance measured against idle-GPU cost.

Next: **[10.3 · Control-plane HA](03-control-plane-ha.md)** — you now know how a single etcd member fails and recovers; the next lesson grows this to a 3-node quorum behind a VIP and adds the version-skew-safe upgrade order that keeps that quorum intact while you patch it.

## References & further reading

**Primary sources**
- **Operating etcd clusters (Kubernetes docs)** — <https://kubernetes.io/docs/tasks/administer-cluster/configure-upgrade-etcd/> — snapshot/restore, resource sizing, and the stacked-kubeadm restore procedure. Read for: the authoritative k8s-side restore steps behind Drill 2.
- **etcd maintenance guide** — <https://etcd.io/docs/latest/op-guide/maintenance/> — history compaction, defrag, the space quota, and the alarm/disarm flow. Read for: grounding Drill 1 and the compaction-vs-defrag distinction.
- **etcd disaster recovery guide** — <https://etcd.io/docs/latest/op-guide/recovery/> — snapshot restore and rebuilding a cluster from one member. Read for: the exact restore commands and multi-member reinit.
- **etcd CHANGELOG-3.6.md** — <https://github.com/etcd-io/etcd/blob/main/CHANGELOG/CHANGELOG-3.6.md> — the authoritative, directly-verified source for the `etcdctl`→`etcdutl` tooling split (PR #13809: removal of `etcdctl snapshot restore`, plus `snapshot status` and offline `defrag` moving to `etcdutl`). Read for: the precise 3.5-vs-3.6 command differences before you trust a runbook.

**Real-world engineering blogs**
- **"Kubernetes ETCD Out of Space on kOps: A Real-Life Incident and Recovery"** — <https://medium.com/@caue._/kubernetes-etcd-out-of-space-on-kops-a-real-life-incident-and-recovery-a1857f3e0998> — what it shows: a real production NOSPACE incident worked through the same compact/defrag/disarm sequence taught here.
- **CNCF "The Kubernetes Surgeon's Handbook: Precision Recovery from etcd Snapshots"** — <https://www.cncf.io/blog/2025/05/08/the-kubernetes-surgeons-handbook-precision-recovery-from-etcd-snapshots/> — what it shows: surgical, narrower-scope snapshot recovery as an alternative to a full DR runbook.
- **CNCF "Making etcd incidents easier to debug in production Kubernetes"** — <https://www.cncf.io/blog/2026/03/12/making-etcd-incidents-easier-to-debug-in-production-kubernetes/> — what it shows: purpose-built `etcd-diagnosis` tooling for triaging real production etcd incidents faster than manual metric-reading.

**Deeper dives**
- **etcd 3.6 announcement blog** — <https://etcd.io/blog/2025/announcing-etcd-3.6/> — the readable companion to the CHANGELOG above; a narrative tour of what's new in 3.6 beyond the tooling split.
