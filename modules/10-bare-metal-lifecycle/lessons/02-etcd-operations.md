---
lesson: "10.2"
title: "etcd operations: disk, quorum, and the 2am restore"
module: "10"
concept: "etcd operations: disk, quorum, and the 2am restore"
status: not-started
est_time: "8h"
artifacts: []
---

# 10.2 · etcd operations: disk, quorum, and the 2am restore

> **Concept.** You know what etcd is — now you own its disk, its quorum, its NOSPACE alarm, and its disaster-recovery runbook.
>
> Module: [🖥️ 10 — Bare metal and cluster lifecycle](../README.md) · Deliverable: [Capex-vs-cloud + KTHW/etcd writeup](../practice/capex-vs-cloud/README.md)

## Why this matters
etcd is the single stateful thing in the cluster and the thing that will actually page you. On EKS/GKE, AWS/Google owned etcd's disk, its backups, and its quorum — you never got that page. On bare metal you do. A full etcd turns the **entire cluster read-only** (no deploys, no scale-ups, no GPU-job scheduling); a lost quorum stops the API cold; a corrupt data dir means restoring from a snapshot under time pressure. This lesson is the runbook you will page yourself with at 2am.

## What's new here
Module 02 taught etcd's **role**: the single source of truth, the only thing the apiserver persists to, watch/revision semantics behind informers. It answered *what etcd is for*.

This lesson is **operations** — nothing here was in 02:
- **Quorum math** and how many failures 3 vs 5 members survive.
- The **`--quota-backend-bytes`** space quota, the **NOSPACE alarm**, and the read-only-cluster failure mode it triggers.
- **Compaction vs defragmentation** — two different reclamations people constantly conflate.
- **Snapshot save / restore**, including the etcd **3.6 tooling split** (save stays in `etcdctl`; **restore moved to `etcdutl`**).
- **Why etcd dies on slow disks** — WAL `fsync` latency — and the exact metric that warns you first.

## Core notes

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
etcd keeps every write in a backend (bbolt/mmap file). To stop unbounded growth it enforces **`--quota-backend-bytes`**: default **2 GiB** if unset/zero, and the **maximum safe value is 8 GiB** (`8*1024*1024*1024`; larger is unsupported/untested). When the backend DB size crosses the quota, etcd raises a cluster-wide **`NOSPACE` alarm** and **refuses all writes**, permitting only reads and the maintenance calls needed to recover.

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
Order matters: **compact → defrag → disarm**. Disarming before you've actually reclaimed space just re-trips the alarm on the next write. Do `defrag` **one member at a time** — defrag blocks that member (it's a stop-the-world rewrite of the DB file), and hitting all members at once can drop you below quorum.

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

The metric that warns you first: **`etcd_disk_wal_fsync_duration_seconds`** (histogram; watch the p99). The companion is **`etcd_disk_backend_commit_duration_seconds`** (bbolt commit). Rule of thumb from the etcd docs: WAL fsync p99 should be **well under ~10ms**; sustained tens-of-ms means the disk can't keep up. Also watch `etcd_server_leader_changes_seen_total` (should be ~flat) and heartbeat-send failures. Fix: **dedicated fast NVMe/SSD for the etcd data dir**, never shared with the container runtime or logs, and raise `--heartbeat-interval`/`--election-timeout` only as a last resort. On a GPU fleet this is real money-versus-reliability: etcd disk is cheap insurance against a fleet-wide stall.

### Snapshots and restore — the 3.6 tooling split
Back up with a point-in-time snapshot:
```bash
ETCDCTL_API=3 etcdctl snapshot save /backup/etcd-$(date +%F-%H%M).db \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key
etcdctl --write-out=table snapshot status /backup/....db   # verify hash/size (deprecated; etcdutl on 3.6)
```
**etcd 3.6 change (Kubernetes 1.33+):** `snapshot save` **stays in `etcdctl`**, but **`snapshot restore` was removed from `etcdctl` and lives in `etcdutl`** (the offline/data-file tool). `snapshot status` likewise moved to `etcdutl`. On 3.5 and earlier, restore was `etcdctl snapshot restore`; hard-coding that in a 3.6 runbook fails.

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
- **Verify** each snapshot (`etcdutl snapshot status` on 3.6+ — check the hash and revision) and periodically **rehearse the restore** on a throwaway VM. An untested restore runbook is the one that fails at 2am.
- Know your **RPO**: the window between snapshots is data you lose on a full-cluster restore (drill 2 shows exactly which objects vanish). For a busy scheduler, 5-minute snapshots mean up to 5 minutes of lost pod/lease state.

### Sizing and the monitoring you actually alert on
etcd is small and latency-bound, not throughput-bound. A large cluster's etcd is happy on a few CPUs, 8–16 GB RAM, and — the part that matters — a **dedicated low-latency SSD/NVMe** for the data dir, isolated from logs and the container runtime. Keep the DB comfortably under the 8 GiB quota ceiling; if you're approaching it, something is leaking (unbounded Events, a controller hot-looping writes, huge ConfigMaps). The alerts worth paging on:

| Signal | Metric | Rough threshold |
|--------|--------|-----------------|
| Disk too slow | `etcd_disk_wal_fsync_duration_seconds` p99 | sustained > ~10ms |
| Backend commit slow | `etcd_disk_backend_commit_duration_seconds` p99 | sustained > ~25ms |
| Election churn | `etcd_server_leader_changes_seen_total` | any sustained rate |
| No leader | `etcd_server_has_leader` | == 0 (page immediately) |
| Approaching quota | `etcd_mvcc_db_total_size_in_bytes` | > ~70% of quota |
| Proposal failures | `etcd_server_proposals_failed_total` | rising |

`etcd_server_has_leader == 0` means quorum is lost — that's the "cluster API is frozen" page. `wal_fsync` p99 climbing is the *early* warning that precedes it, which is why it's the answer to "what warns you first."

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

## Worked example — trace of the two drills on a stacked kubeadm node

**Drill 1 (quota → read-only → recover):** Push junk into etcd until `db size` exceeds a tiny quota you set (`--quota-backend-bytes=$((16*1024*1024))` for testing), e.g. loop `etcdctl put junk/$i <64KB>`. Watch `etcdctl alarm list` flip to `NOSPACE`; simultaneously `kubectl create deploy x --image=nginx` fails with `mvcc: database space exceeded`. Recover: get current revision → `etcdctl compact <rev>` → `etcdctl defrag` → `etcdctl alarm disarm`. Confirm `kubectl apply` succeeds again and `endpoint status` shows `db size` shrunk. Document what the apiserver returned before and after.

**Drill 2 (destroy data dir → restore):** `etcdctl snapshot save` a good backup. Stop the etcd static pod (`mv /etc/kubernetes/manifests/etcd.yaml /tmp`), `rm -rf /var/lib/etcd/member` (the data dir). The apiserver now crashloops (no etcd). Restore: `etcdutl snapshot restore <snap> --data-dir /var/lib/etcd --name <node> --initial-cluster <node>=https://<ip>:2380 --initial-advertise-peer-urls https://<ip>:2380`, then `mv /tmp/etcd.yaml /etc/kubernetes/manifests/`. The kubelet restarts etcd, the apiserver recovers, and `kubectl get` shows exactly the cluster state as of the snapshot revision — anything created after the snapshot is gone. Note the revision gap in your writeup.

## Practice
On a 1- or 3-VM cluster (KVM/multipass/small cloud VMs), stacked kubeadm etcd is fine.

**Drill 1 — fill and recover:** set a tiny `--quota-backend-bytes`, fill past it, observe the NOSPACE alarm and the apiserver going read-only (capture the exact write error and a failing `kubectl apply`), then recover via `compact → defrag → alarm disarm`. Record `endpoint status` `db size` before/after.

**Drill 2 — destroy and restore:** `etcdctl snapshot save`, verify with `etcdutl snapshot status`, create a marker object (`kubectl create cm dr-marker --from-literal=t=$(date +%s)`) *after* the snapshot so you can prove it disappears, destroy the data dir, watch the apiserver crashloop (`crictl logs` on the apiserver container / `journalctl -u kubelet`), restore with **`etcdutl snapshot restore`** (confirm you're on 3.6+ via `etcd --version` / `kubectl version` ≥1.33 — the 3.5 `etcdctl snapshot restore` command is gone), and prove recovery: apiserver healthy, `kubectl get nodes` Ready, and the `dr-marker` ConfigMap **absent** (it was created after the snapshot revision).

**Acceptance:** both failure drills documented with **exact commands, the etcd tool used (and the 3.5-vs-3.6 restore difference), and what the apiserver did** at each step — feeds the deliverable's etcd writeup alongside the KTHW cert table from 10.1.

## Self-check
**(a) 3-member vs 5-member etcd — how many simultaneous member failures does each tolerate, and why?**
**Answer:** 3 tolerates **1**, 5 tolerates **2**. A write needs a majority (quorum = ⌊N/2⌋+1 = 2 and 3); you can lose `N − quorum` members and still form a majority. Even sizes don't help (4 also needs 3, tolerates only 1), which is why you run 3/5/7.

**(b) Compaction vs defragmentation — what does each reclaim?**
**Answer:** **Compaction** discards superseded MVCC key **history** below a revision — frees logical space inside the DB file but doesn't shrink it (auto every 5 min via the apiserver). **Defragmentation** rewrites the backend file to **return freed pages to the filesystem**, actually shrinking `db size` — disruptive, manual, one member at a time.

**(c) Why does etcd hate slow disks — what latency dominates, what metric warns you?**
**Answer:** Every Raft commit must `fsync` the WAL to disk before it's acknowledged, so **WAL fsync latency dominates** — slow disks stall commits, miss heartbeats, and churn leader elections. First warning metric: **`etcd_disk_wal_fsync_duration_seconds`** (watch p99; keep it well under ~10ms), backed up by `etcd_disk_backend_commit_duration_seconds` and `etcd_server_leader_changes_seen_total`.

## Resources
1. **Operating etcd clusters (Kubernetes docs)** — https://kubernetes.io/docs/tasks/administer-cluster/configure-upgrade-etcd/ — snapshot/restore, resource sizing, and the stacked-kubeadm restore procedure. **Deep** (this is your drill-2 reference). Why: authoritative for the k8s-side restore steps.
2. **etcd maintenance guide** — https://etcd.io/docs/latest/op-guide/maintenance/ — history compaction, defrag, the space quota, and the alarm/disarm flow. **Deep.** Why: grounds drill 1 and the compaction-vs-defrag distinction; note the 3.6 `etcdutl` split for `snapshot status`.
3. **etcd disaster recovery guide** — https://etcd.io/docs/latest/op-guide/recovery/ — snapshot restore and rebuilding a cluster from one member; note **restore is `etcdutl` on 3.6+**. **Deep.** Why: the exact restore commands and multi-member reinit.
