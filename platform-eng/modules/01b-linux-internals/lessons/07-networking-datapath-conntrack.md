---
lesson: "01b.7"
title: "The network datapath: netfilter, conntrack, and NAT at scale"
module: "01b"
concept: "The network datapath: netfilter, conntrack, and NAT at scale"
status: not-started
est_time: "6h"
prev: "06-hugepages-thp-numa.md"
next: "08-ebpf.md"
artifacts: []
sources: 18
---

# 01b.7 · The network datapath: netfilter, conntrack, and NAT at scale

> **Concept.** Every NATed pod flow leaves a fingerprint in the kernel's connection-tracking table; when that table fills, a busy node drops packets silently — and eBPF datapaths exist largely to escape it.
>
> Module: [🐧 01b — Linux systems internals](../README.md) · Deliverable: [Anatomy of a Container](../practice/anatomy-of-a-container/README.md)

## Where this fits

[Lesson 06](06-hugepages-thp-numa.md) was about a *memory-side* shared resource under fleet load: hugepages, THP, and NUMA locality, where the failure mode is TLB pressure and cross-socket latency quietly stealing throughput from a GPU or a NIC. This lesson is the network-side sibling of the same pattern — a node-local kernel resource, shared by every pod on the box, that degrades invisibly until it doesn't. Here the resource is the `nf_conntrack` table and the mechanism is netfilter hooks plus NAT.

It also closes a hole [lesson 04](04-psi.md) deliberately left open. PSI covers CPU, memory, I/O, and optionally IRQ. **There is no `/proc/pressure/net`.** A node that has stopped accepting new connections because its conntrack table is full reports perfectly clean pressure on every domain, a perfectly healthy load average, and idle CPU. The saturation instrument you learned to trust two lessons ago is structurally blind to this failure, which is exactly why it gets its own lesson and its own counters.

By the end you will be able to trace the ordered list of netfilter hooks a packet actually traverses and say which subsystem registered at which priority; explain why NAT cannot exist without a conntrack entry per flow; size a conntrack table from a connection rate and a timeout; read `conntrack -S` and know which counter means "table full" and which means something completely different; and explain what kube-proxy's iptables mode, its IPVS mode, its nftables mode, and Cilium's eBPF datapath each do to the per-packet cost and to the table.

That last point is the hinge into the next lesson. **[08 — eBPF](08-ebpf.md)** unpacks the general-purpose mechanism (verifier, JIT, hooks, maps) that Cilium's socket load balancer and its BPF conntrack map are built from. You leave this lesson knowing *that* an eBPF datapath sidesteps the conntrack bottleneck and *what* it replaces; the next lesson explains how eBPF does anything at all.

## Why this matters

On a GPU cluster the network *is* the workload, but not in the way people assume. The gradient traffic — NCCL all-reduce over RoCE or InfiniBand — bypasses the host TCP/IP stack entirely and never touches netfilter. Everything else does:

- **Egress fan-out.** Dataloaders pulling shards out of S3/GCS, checkpoint uploads, metrics push, log shipping, image pulls, control-plane calls. Each TCP connection is one conntrack entry, and because pods egress via source NAT to the node IP, that entry lives in a table shared by every pod on the node.
- **Inference serving.** A node running many short-lived request/response flows — an inference gateway, a token-streaming endpoint, a batch-embedding service — is a conntrack *generator*. The arithmetic in §9 shows how few requests per second it takes to fill the kernel's default table.
- **DNS.** Every UDP query from a pod that leaves the node creates a conntrack entry, and UDP is where the nastiest conntrack race lives (§6).

When the table fills, the kernel logs `nf_conntrack: table full, dropping packet` and starts dropping packets that would need a *new* entry. The user-visible symptom is maddening and misleading: existing connections keep working, new ones hang and time out, intermittently, under load, on your busiest nodes — precisely when a training job ramps its data pipeline. Nobody's dashboard shows it, because utilization is fine, PSI is clean, and the NIC counters show no drops. The drop happened in software, in a hook, on a table nobody graphs.

Knowing this mechanism is the difference between "the network is flaky, escalate to netops" and "conntrack is at 98% of `nf_conntrack_max` on nodes 14 and 27, here is the counter that proves it, here is the tactical sysctl, and here is why the architectural fix is a datapath change rather than another bump." That is a cost-and-observability differentiator: you catch it on a dashboard before it pages, and at a staff level you can argue for the structural fix instead of the fourth sysctl in eighteen months.

## What's new here (calibration)

Per the module README's calibration, you already know shell and networking basics, Kubernetes Services in the abstract, and how to squint at `iptables -t nat -L` or `conntrack -L`. None of that is re-taught. What is genuinely new:

- **The hook architecture with real names and real priorities.** Not "packets go through netfilter" but the five `nf_inet_hooks` constants, the priority numbers that order the subsystems registered on each one, and why `raw` at −300 is the only place you can opt a flow *out* of tracking.
- **The mechanical coupling between NAT and conntrack.** NAT is not a separate feature that happens to use conntrack; the translation is *stored in the conntrack entry's reply tuple*. One conntrack entry per NATed flow is not a tunable, it is a structural consequence of where the reverse mapping lives.
- **Real sizing, from source.** How `nf_conntrack_buckets` and `nf_conntrack_max` are actually computed at module init, what `kube-proxy` then overrides them to, what a single entry costs in kernel memory (measured, not guessed), and what happens between "the table is full" and "the packet is dropped" — there is an eviction attempt in between, and a garbage collector that changes behaviour at 95%.
- **The counters, with correct semantics.** `drop`, `insert_failed`, `early_drop`, `clash_resolve`, `chaintoolong` mean five different things. Most write-ups treat `drop` and `insert_failed` as synonyms. They are not, and the difference tells you which of two completely different incidents you are in.
- **The lookup-cost difference between kube-proxy modes**, drawn rather than asserted: a linear chain walk versus a hash lookup versus a verdict map, and what each does (and does not do) to the conntrack table.

## Core concepts

### 1. Netfilter: the problem, the five hooks, and the priority ladder

**The problem.** The kernel's IP stack is a pipeline: receive a frame, parse the header, decide whether the packet is for this host or for someone else, deliver or forward it, transmit. Firewalling, address translation, connection tracking, packet mangling, and logging all need to interpose on that pipeline. Without a shared framework, each of them would have to patch the receive path directly, and they would fight over ordering.

Netfilter is that framework, and it is deliberately minimal: **at five fixed points in the IPv4/IPv6 stack, the kernel calls out to a list of registered functions, in priority order, and each one returns a verdict.** The five points are the `nf_inet_hooks` enum (`include/uapi/linux/netfilter.h`):

| Hook | Value | Where in the stack | What reaches it |
|---|---|---|---|
| `NF_INET_PRE_ROUTING` | 0 | after the packet is parsed, **before** the routing decision | every received packet |
| `NF_INET_LOCAL_IN` | 1 | after routing said "this is for us" | packets destined to a local socket |
| `NF_INET_FORWARD` | 2 | after routing said "this is for someone else" | packets being routed through |
| `NF_INET_LOCAL_OUT` | 3 | after a local socket generated a packet | locally originated traffic |
| `NF_INET_POST_ROUTING` | 4 | just before handing to the device layer | everything leaving, forwarded or local |

There is also `NF_INET_INGRESS` (value 5, aliased to `NF_INET_NUMHOOKS`) for the nftables ingress hook, and a separate `nf_dev_hooks` enum with `NF_NETDEV_INGRESS` and `NF_NETDEV_EGRESS` that sit at the device layer rather than the IP layer.

The verdicts a hook function may return are equally short: `NF_DROP` (0), `NF_ACCEPT` (1), `NF_STOLEN` (2), `NF_QUEUE` (3), `NF_REPEAT` (4). `NF_DROP` frees the packet and returns; nothing further in the chain runs, and **nothing tells the sender**. That silence is the root of the whole diagnostic problem in this lesson: a conntrack-exhaustion drop produces no ICMP, no RST, no NIC counter — the SYN simply evaporates and the client waits out its connect timeout.

**Ordering is by integer priority, lower first** (`enum nf_ip_hook_priorities`, `include/uapi/linux/netfilter_ipv4.h`). These numbers are the actual API; memorising a handful of them turns "which happens first, NAT or filtering?" into arithmetic:

| Priority | Constant | Registered by |
|---|---|---|
| −450 | `NF_IP_PRI_RAW_BEFORE_DEFRAG` | raw table, pre-defrag |
| −400 | `NF_IP_PRI_CONNTRACK_DEFRAG` | IP defragmentation (conntrack needs whole packets) |
| −300 | `NF_IP_PRI_RAW` | the `raw` table — **the only place before tracking** |
| −225 | `NF_IP_PRI_SELINUX_FIRST` | SELinux |
| −200 | `NF_IP_PRI_CONNTRACK` | **conntrack lookup / creation** |
| −150 | `NF_IP_PRI_MANGLE` | the `mangle` table |
| −100 | `NF_IP_PRI_NAT_DST` | the `nat` table, DNAT direction |
| 0 | `NF_IP_PRI_FILTER` | the `filter` table |
| 50 | `NF_IP_PRI_SECURITY` | the `security` table |
| 100 | `NF_IP_PRI_NAT_SRC` | the `nat` table, SNAT direction |
| 300 | `NF_IP_PRI_CONNTRACK_HELPER` | protocol helpers (FTP, SIP, …) |
| `INT_MAX` | `NF_IP_PRI_CONNTRACK_CONFIRM` | **conntrack confirmation — always last** |

Three structural facts fall straight out of this table, and they are the ones worth carrying:

1. **Defrag runs before conntrack (−400 < −200).** Conntrack keys on layer-4 ports, which only exist in the first fragment, so it cannot function on fragments. The kernel reassembles first.
2. **`raw` runs before conntrack (−300 < −200).** That is why `iptables -t raw -A PREROUTING -j CT --notrack` (or `-j NOTRACK` on old kernels) is the *only* way to make a flow skip the table entirely. Every other table runs after the entry already exists.
3. **Conntrack confirmation is at `INT_MAX`, i.e. dead last, on `POST_ROUTING` and `LOCAL_IN`.** This is the single most under-appreciated fact about conntrack and it drives §2: an entry is created early but **not inserted into the hash table until the packet has survived the entire hook chain**. If a firewall rule drops the packet at priority 0, no entry is ever inserted.

Conntrack's own hook registration confirms it — four entries, verbatim in structure from `net/netfilter/nf_conntrack_proto.c`:

| Hook function | Hook | Priority |
|---|---|---|
| `ipv4_conntrack_in` | `NF_INET_PRE_ROUTING` | `NF_IP_PRI_CONNTRACK` (−200) |
| `ipv4_conntrack_local` | `NF_INET_LOCAL_OUT` | `NF_IP_PRI_CONNTRACK` (−200) |
| `nf_confirm` | `NF_INET_POST_ROUTING` | `NF_IP_PRI_CONNTRACK_CONFIRM` (`INT_MAX`) |
| `nf_confirm` | `NF_INET_LOCAL_IN` | `NF_IP_PRI_CONNTRACK_CONFIRM` (`INT_MAX`) |

Two entry points (one for received packets, one for locally generated), two confirmation points (one for packets that leave, one for packets that terminate locally).

Here is the whole journey. This is the diagram to hold in your head for the rest of the lesson.

```
              THE PACKET'S JOURNEY — one received packet, ingress to socket
              (hook names and priorities are the kernel's own constants)

   wire
    │
    ▼
 ┌─────────┐   DMA into rx ring, then NAPI poll in softirq context
 │   NIC   │──────────────────────────────────────────────────────┐
 └─────────┘                                                      │
                                                                  ▼
                                                       ┌──────────────────────┐
                                    XDP_DROP/TX/PASS ◀─│  XDP  (driver hook)  │  no skb yet
                                                       └──────────┬───────────┘
                                                                  │ XDP_PASS
                                                                  ▼
                                                       ┌──────────────────────┐
                                                       │  build_skb()         │
                                                       └──────────┬───────────┘
                                                                  ▼
                                                       ┌──────────────────────┐
                                                       │ tc ingress (clsact)  │  eBPF/tc lives here
                                                       └──────────┬───────────┘
                                                                  ▼
                                                       ┌──────────────────────┐
                                                       │ NF_NETDEV_INGRESS    │  nft ingress hook
                                                       └──────────┬───────────┘
                                                                  ▼
                                                       ┌──────────────────────┐
                                                       │  ip_rcv()            │
                                                       └──────────┬───────────┘
                                                                  ▼
 ╔══════════════════════════════ NF_INET_PRE_ROUTING ═════════════════════════════╗
 ║  −400  conntrack defrag        reassemble fragments                             ║
 ║  −300  raw table               ── `-j CT --notrack` HERE = skip tracking ──     ║
 ║  −200  CONNTRACK               ┌───────────────────────────────────────────┐    ║
 ║                                │ build tuple from (saddr,daddr,proto,      │    ║
 ║                                │ sport,dport)  →  hash  →  lookup          │    ║
 ║                                │  HIT  : attach existing ct, set ctinfo    │    ║
 ║                                │  MISS : allocate nf_conn (256 B slab),    │    ║
 ║                                │         mark UNCONFIRMED, ctinfo = NEW    │    ║
 ║                                │         ← this is where "table full" hits │    ║
 ║                                └───────────────────────────────────────────┘    ║
 ║  −150  mangle                                                                   ║
 ║  −100  nat (DNAT)              only consulted if ctinfo == NEW; the chosen       ║
 ║                                translation is written into the REPLY TUPLE       ║
 ╚═══════════════════════════════════════╤═════════════════════════════════════════╝
                                         ▼
                              ┌─────────────────────┐
                              │  ROUTING DECISION   │  fib_lookup on the (possibly
                              │  ip_route_input()   │  already DNATed) destination
                              └───────┬─────────┬───┘
                     for us ──────────┘         └────────── for someone else
                          ▼                                        ▼
 ╔══════════ NF_INET_LOCAL_IN ══════════╗       ╔════════ NF_INET_FORWARD ════════╗
 ║    0  filter (INPUT)                 ║       ║    0  filter (FORWARD)          ║
 ║  100  nat (SNAT, rare)               ║       ╚════════════════╤════════════════╝
 ║  MAX  nf_confirm ── INSERT INTO HASH ║                        ▼
 ╚══════════════════╤═══════════════════╝       ╔═══════ NF_INET_POST_ROUTING ════╗
                    ▼                           ║  −150  mangle                   ║
              ┌───────────┐                     ║   100  nat (SNAT/MASQUERADE)    ║
              │  socket   │                     ║   MAX  nf_confirm ── INSERT ──▶ ║
              │ (tcp_v4_rcv)                    ╚════════════════╤════════════════╝
              └───────────┘                                      ▼
                                                        tc egress → driver → wire

  Locally generated packets enter at NF_INET_LOCAL_OUT (conntrack −200, nat DST −100,
  filter 0), take the routing decision, and rejoin at NF_INET_POST_ROUTING.

  THE TWO FACTS THIS PICTURE ENCODES:
   (a) the table lookup happens ONCE per packet, at −200, before any rule is evaluated;
   (b) the entry is INSERTED only at INT_MAX, after every rule has accepted the packet.
```

### 2. What a conntrack entry actually is

**The problem it solves.** A stateless firewall can only match on fields in the packet in front of it. To express "allow replies to connections we started," it needs memory. To translate addresses, it needs *more* memory: having rewritten a pod's source address to the node's address on the way out, something must remember how to rewrite it back on the way in. Connection tracking is that memory, and NAT is its first and biggest customer.

**The data structure.** An entry is a `struct nf_conn` allocated from a dedicated slab cache created at module init with `kmem_cache_create("nf_conntrack", sizeof(struct nf_conn), …)`. You can read its real size on your own kernel:

```bash
$ grep -E '^(# name|nf_conntrack )' /proc/slabinfo
# name            <active_objs> <num_objs> <objsize> <objperslab> <pagesperslab> : ...
nf_conntrack           0      0    256   16    1 : tunables    0    0    0 : ...
```

*(Captured on Linux 6.18.5, x86-64.)* **256 bytes per entry** on this build — read the `objsize` column on yours, because it moves with config options and kernel version. NATed flows additionally carry a `struct nf_conn_nat` extension, which is tiny (a union plus an `int masq_index`), allocated in the entry's extension area. For capacity planning, budget ~256–400 B per tracked flow and verify.

The heart of the entry is **two tuples**, one per direction, stored in `ct->tuplehash[IP_CT_DIR_ORIGINAL]` and `ct->tuplehash[IP_CT_DIR_REPLY]`. A tuple is `(src addr, dst addr, layer-3 proto, src port, dst port, layer-4 proto)`. Both tuples are hashed into **the same global hash table**, which is why the kernel documentation warns that entries occupy the table twice:

> `nf_conntrack_max` … "connection tracking entries are added to the table twice — once for the original direction and once for the reply direction (i.e., with the reversed address). This means that with default settings a maxed-out table will have an average hash chain length of 2, not 1." (`Documentation/networking/nf_conntrack-sysctl.rst`)

**The status bits** that matter operationally:

| Bit | Meaning | Why you care |
|---|---|---|
| `IPS_CONFIRMED` | the entry has been inserted into the hash table | before this it exists only on the `skb` |
| `IPS_SEEN_REPLY` | traffic has been observed in both directions | `conntrack -L` prints `[UNREPLIED]` when unset |
| `IPS_ASSURED` | the flow is considered established and worth keeping | **`ASSURED` entries are never early-dropped (§5)** |
| `IPS_SRC_NAT` / `IPS_DST_NAT` | NAT was applied in that direction | the reply tuple is not merely the reverse of the original |

For TCP, `IPS_ASSURED` is set when a valid ACK is seen in `ESTABLISHED` after `SYN_RECV` (`nf_conntrack_proto_tcp.c`) — i.e. once the three-way handshake has completed and data can flow. A half-open connection is *not* assured, which is exactly what makes it evictable under pressure.

**The lifecycle**, which is the piece most explanations skip:

```
   CONNTRACK ENTRY LIFECYCLE — where each transition happens, and what can kill it

  packet arrives
       │
       ▼
  ┌────────────────────┐  PRE_ROUTING/-200 or LOCAL_OUT/-200
  │  hash lookup       │
  └──┬──────────────┬──┘
     │ HIT          │ MISS
     │              ▼
     │   ┌──────────────────────────────┐
     │   │ __nf_conntrack_alloc()        │
     │   │  count = atomic_inc(cnet)     │
     │   │  if count > nf_conntrack_max: │
     │   │     early_drop() ── scan 8    │──fail──▶ ratelimited dmesg:
     │   │        buckets for a non-     │          "nf_conntrack: table full,
     │   │        ASSURED victim         │           dropping packet"
     │   │     success ▶ early_drop++    │          drop++ ; NF_DROP ; PACKET GONE
     │   │  kmem_cache_alloc(256 B)      │
     │   └───────────┬──────────────────┘
     │               ▼
     │        ┌──────────────┐   lives ONLY on skb->_nfct; invisible to
     │        │ UNCONFIRMED  │   `conntrack -L`; visible via `conntrack -L unconfirmed`
     │        └──────┬───────┘
     │               │  nat table at -100 / +100 may now rewrite the REPLY tuple
     │               ▼
     │     ...all remaining hooks: mangle, filter, security...
     │               │
     │        filter -j DROP ──────────────▶ entry freed, never inserted, no cost
     │               ▼
     │   ┌────────────────────────────────┐  POST_ROUTING or LOCAL_IN, priority INT_MAX
     │   │ __nf_conntrack_confirm()        │
     │   │  · re-check for a clashing      │──clash, protocol disallows──▶ drop++ AND
     │   │    tuple inserted meanwhile     │                              insert_failed++
     │   │  · walk chain; if > ~50–80      │──chaintoolong++ , insert_failed++
     │   │    entries, refuse              │
     │   │  · ct->timeout += now           │
     │   │  · insert BOTH tuplehashes      │
     │   └───────────┬────────────────────┘
     ▼               ▼
  ┌───────────────────────────┐
  │        CONFIRMED          │  now in the hash, counted in nf_conntrack_count,
  │  timeout = per-state value│  now visible to `conntrack -L`
  └───────────┬───────────────┘
              │  every subsequent packet refreshes ct->timeout
              │
      ┌───────┴─────────┬──────────────────┬────────────────────┐
      ▼                 ▼                  ▼                    ▼
  timeout fires    gc_worker scan     early_drop victim    conntrack -D / -F
  (lazy: checked   (every 1–60 s,     (only if !ASSURED)   (explicit delete)
   on lookup and   ≤10 ms per run)
   by the gc)
      └─────────────────┴──────────────────┴────────────────────┘
                                  ▼
                             entry freed
```

**Read a real entry.** `conntrack -L` dumps the confirmed table; the field order is fixed by `libnetfilter_conntrack`'s default formatter (`__snprintf_conntrack_default`): L4 protocol name, protocol number, remaining timeout in seconds, TCP state, original tuple, `[UNREPLIED]` if the reply has not been seen, reply tuple, `[ASSURED]` if set, then `mark=`, and `use=` (the refcount).

```
$ conntrack -L -p tcp --dport 443 2>/dev/null
tcp      6 86391 ESTABLISHED src=10.244.3.7 dst=52.94.1.2 sport=51042 dport=443 \
         src=52.94.1.2 dst=192.0.2.10 sport=443 dport=51042 [ASSURED] mark=0 use=1
tcp      6 118 TIME_WAIT src=10.244.3.7 dst=52.94.1.2 sport=50988 dport=443 \
         src=52.94.1.2 dst=192.0.2.10 sport=443 dport=50988 [ASSURED] mark=0 use=1
tcp      6 111 SYN_SENT src=10.244.3.7 dst=52.94.1.9 sport=51104 dport=443 \
         [UNREPLIED] src=52.94.1.9 dst=192.0.2.10 sport=443 dport=51104 mark=0 use=1
conntrack v1.4.7 (conntrack-tools): 3 flow entries have been shown.
```

*(Representative transcript; the field order and markers are exact.)* Read line one field by field:

- `tcp 6` — the L4 protocol by name and number.
- `86391` — seconds until this entry expires. Not 431 999: this node's `nf_conntrack_tcp_timeout_established` is 86 400, because **kube-proxy set it** (§7). On a machine with no kube-proxy you would see a number just under 432 000.
- `ESTABLISHED` — conntrack's own TCP state machine, which is related to but not identical to the socket's state.
- Original tuple: pod `10.244.3.7:51042` → `52.94.1.2:443`.
- Reply tuple: `52.94.1.2:443` → **`192.0.2.10:51042`**. The pod's address is *gone*; the destination of the reply is the node's address. **That asymmetry is the masquerade.** The reply tuple is not the reverse of the original — it is the reverse *after translation*, which is precisely the stored mapping.
- `[ASSURED]` — handshake completed; this entry is protected from early drop.
- `use=1` — refcount.

Line three is the interesting one: a `SYN_SENT` entry, `[UNREPLIED]`, 111 seconds left of the 120-second `nf_conntrack_tcp_timeout_syn_sent`. **A connection attempt that never got an answer still costs a table slot for two minutes.** Under a retry storm that is the dominant term in your occupancy.

Companion commands, all from `conntrack-tools`:

| Command | What it does | Cost |
|---|---|---|
| `conntrack -L` | dump the confirmed table | full dump — expensive on a large table |
| `conntrack -L unconfirmed` | dump entries not yet inserted | tiny |
| `conntrack -C` | print the current count | one netlink round trip |
| `conntrack -S` | per-CPU statistics | cheap, and the one you actually alert on |
| `conntrack -E` | stream `[NEW]`/`[UPDATE]`/`[DESTROY]` events | continuous, no dump cost |
| `conntrack -D -p udp --dport 53` | delete matching entries | targeted surgery |
| `conntrack -F` | flush the whole table | **breaks every NATed flow on the node** |

`conntrack -E` is the right instrument for watching churn on a hot node: `-L` copies the entire table across netlink every time you invoke it, so a one-second poll loop on a 500 k-entry table is itself a load generator. Event mode subscribes to ctnetlink and prints state changes as they happen. (It depends on `nf_conntrack_events`, default `2` = "auto": the event extension is allocated only when a userspace listener exists.)

### 3. NAT is implemented *on* conntrack — the coupling, in detail

**The problem.** Pod addresses (`10.244.0.0/16` in a typical CNI) are not routable outside the cluster. For a pod to reach S3, something must rewrite the source address to the node's routable address on the way out, and rewrite the destination back to the pod's address on the way in. The outbound rewrite is easy. The inbound rewrite requires knowing, for a packet arriving at `192.0.2.10:51042`, which pod and which port that maps to.

**The mechanism.** Netfilter does not keep a separate NAT table. It stores the translation **inside the conntrack entry's reply tuple**, and this is the whole trick:

```
  BEFORE NAT — the entry as created at PRE_ROUTING/-200 (or LOCAL_OUT)

    original tuple :  10.244.3.7:51042  →  52.94.1.2:443
    reply tuple    :  52.94.1.2:443     →  10.244.3.7:51042      (plain inversion)

  The nat table at POST_ROUTING/+100 runs `-j MASQUERADE`, which calls
  nf_nat_setup_info(ct, range, NF_NAT_MANIP_SRC). That does NOT edit the packet
  first — it EDITS THE REPLY TUPLE, then derives the packet edit from it:

  AFTER NAT

    original tuple :  10.244.3.7:51042  →  52.94.1.2:443          (unchanged)
    reply tuple    :  52.94.1.2:443     →  192.0.2.10:51042       (rewritten!)
                                            ▲          ▲
                                            │          └── source port, kept if free
                                            └───────────── node IP chosen by MASQUERADE
                                                           from the outbound route

  Now every packet is handled by ONE rule, applied in both directions:
    outbound packet matches the ORIGINAL tuple → rewrite src to the reply tuple's dst
    inbound  packet matches the REPLY    tuple → rewrite dst to the original tuple's src

  The reply tuple IS the NAT mapping. There is nowhere else it could live.
```

Three consequences follow, and they are the ones people get wrong:

**(a) The `nat` table is consulted exactly once per flow.** `iptables -t nat` rules are only evaluated when `ctinfo == IP_CT_NEW` — i.e., on the first packet, before the entry is confirmed. Every subsequent packet is translated by conntrack replaying the stored tuples, never by re-walking the rules. This is the efficiency win and the liability in one sentence: **rule-evaluation cost is per-flow, state cost is per-flow, and the state is what runs out.**

**(b) One flow is exactly one entry, always.** You cannot have NAT without an entry, cannot share entries between flows, and cannot make the entry smaller. "Reduce conntrack usage" therefore means exactly one of three things: fewer flows (connection reuse), shorter-lived entries (timeouts), or flows that are not NATed and not tracked (`--notrack`, direct routing, or a different datapath).

**(c) The source port is part of the tuple, so NAT can run out of ports.** `MASQUERADE` prefers to keep the original source port; if `(node IP, port)` collides with an existing tuple to the same destination it must pick another. With many pods behind one node address talking to the *same* destination `(IP, port)` pair, the space is the ~28 000 ports in `net.ipv4.ip_local_port_range` (default `32768 60999`), per destination endpoint. kube-proxy's masquerade rule uses **`--random-fully`**, which randomises the whole port selection rather than only the high bits:

```
-A KUBE-POSTROUTING -m mark ! --mark 0x4000/0x4000 -j RETURN
-A KUBE-POSTROUTING -j MARK --xor-mark 0x4000
-A KUBE-POSTROUTING -m comment --comment "kubernetes service traffic requiring SNAT" \
   -j MASQUERADE --random-fully
```

*(Reproduced from `pkg/proxy/iptables/proxier.go`; the mark value comes from `--iptables-masquerade-bit`, default 14, i.e. `0x4000`.)* Note the structure: the chain returns immediately unless the packet was explicitly marked for masquerading upstream (`KUBE-MARK-MASQ` sets the bit), clears the mark so a packet re-entering the stack is not masqueraded twice, then masquerades. `--random-fully` exists because sequential port allocation makes collisions — and therefore `insert_failed` events — far more likely under concurrency.

### 4. Timeouts: the state machine that decides how long an entry lives

Occupancy is rate × lifetime, and lifetime is set by a per-state timeout that resets on every packet. These are the kernel defaults, from `Documentation/networking/nf_conntrack-sysctl.rst` cross-checked against the `tcp_timeouts[]` array in `net/netfilter/nf_conntrack_proto_tcp.c`:

| sysctl (`net.netfilter.` prefix) | Default | Applies to |
|---|---|---|
| `nf_conntrack_tcp_timeout_syn_sent` | 120 s | SYN sent, no reply yet |
| `nf_conntrack_tcp_timeout_syn_recv` | 60 s | SYN+ACK seen |
| `nf_conntrack_tcp_timeout_established` | **432 000 s (5 days)** | established connection |
| `nf_conntrack_tcp_timeout_fin_wait` | 120 s | FIN sent |
| `nf_conntrack_tcp_timeout_close_wait` | 60 s | FIN received, local side still open |
| `nf_conntrack_tcp_timeout_last_ack` | 30 s | waiting for final ACK |
| `nf_conntrack_tcp_timeout_time_wait` | **120 s** | after close |
| `nf_conntrack_tcp_timeout_close` | 10 s | closed |
| `nf_conntrack_tcp_timeout_max_retrans` | 300 s | after `tcp_max_retrans` (3) unacked retransmits |
| `nf_conntrack_tcp_timeout_unacknowledged` | 300 s | data sent, never acknowledged |
| `nf_conntrack_udp_timeout` | **30 s** | UDP, single exchange |
| `nf_conntrack_udp_timeout_stream` | 120 s | UDP seen in both directions repeatedly |
| `nf_conntrack_icmp_timeout` | 30 s | ICMP echo |
| `nf_conntrack_generic_timeout` | 600 s | protocols with no tracker |

Verify on any host:

```bash
$ sysctl net.netfilter.nf_conntrack_tcp_timeout_established \
        net.netfilter.nf_conntrack_tcp_timeout_time_wait \
        net.netfilter.nf_conntrack_udp_timeout
net.netfilter.nf_conntrack_tcp_timeout_established = 432000
net.netfilter.nf_conntrack_tcp_timeout_time_wait = 120
net.netfilter.nf_conntrack_udp_timeout = 30
```

*(Captured on a 4-core / 16 GB Linux 6.18.5 host with no kube-proxy running.)*

**The two defaults that dominate GPU-node behaviour:**

*5 days for `ESTABLISHED`.* An idle-but-open connection holds a slot for almost a week. That is deliberate — conntrack cannot see application-level liveness, and expiring a live-but-quiet connection silently breaks it, since the reply would arrive with no entry to translate it. It also means long-lived connection pools are essentially permanent occupancy. kube-proxy reduces this to 24 hours by default (§7).

*120 s for `TIME_WAIT`.* This is the one that makes churn expensive. A connection that lived 40 ms still holds its slot for two more minutes after closing. A workload issuing 2 000 short connections/s therefore parks roughly 240 000 entries in `TIME_WAIT` steady-state — from a workload whose *concurrent* connection count might be 80.

**Occupancy arithmetic — Little's law, applied to a table.** In steady state, entries = arrival rate × mean lifetime:

```
    N  =  λ × (d + T)

    N  = steady-state conntrack entries
    λ  = new connections per second
    d  = mean connection duration (seconds)
    T  = post-close residency: TIME_WAIT (120 s) for the side that closes first,
         or FIN_WAIT/CLOSE_WAIT/LAST_ACK for other close patterns

  Worked, for a dataloader opening one HTTPS connection per object fetched:
    λ = 800 conn/s, d = 0.05 s, T = 120 s
    N = 800 × 120.05 ≈ 96 040 entries

  Note what dominates: the 0.05 s of useful work contributes 40 entries.
  The 120 s of TIME_WAIT contributes 96 000. THE TABLE IS ~99.96% DEAD FLOWS.

  Inverting to find the ceiling — the rate at which a table of size M saturates:
    λ_max = M / (d + T)  ≈  M / T   when d ≪ T

    kernel default, no kube-proxy:  M = 262 144  →  λ_max ≈ 2 184 conn/s
    kube-proxy on a 16-core node:   M = 524 288  →  λ_max ≈ 4 369 conn/s
    kube-proxy on a 128-core node:  M = 4 194 304 →  λ_max ≈ 34 952 conn/s  (see §7)
```

**The trade-off in shortening `TIME_WAIT`.** Dropping `nf_conntrack_tcp_timeout_time_wait` from 120 s to 30 s cuts steady-state occupancy by 4×, which is the single most effective knob. The risk is tuple reuse: if the same `(src IP, src port, dst IP, dst port)` four-tuple is reused before a delayed segment from the previous connection has drained from the network, that segment can be matched to the new flow's entry. TCP's own `TIME_WAIT` exists for the same reason (2×MSL). Reducing conntrack's timeout below the network's realistic maximum segment lifetime is a real correctness risk, not a free win — it is defensible on a datacentre-internal path with sub-millisecond RTT and indefensible on a WAN path. **Reduce connection count first; shorten timeouts second.**

### 5. Sizing, eviction, and what actually happens at the ceiling

**How the defaults are computed.** At module init (`nf_conntrack_init_start()` in `net/netfilter/nf_conntrack_core.c`), if `hashsize` was not given as a module parameter:

```
  buckets = (total_RAM_bytes / 16384) / sizeof(struct hlist_head)     # /8 on 64-bit
  if 64-bit and RAM > 4 GiB:  buckets = 262144
  elif RAM > 1 GiB:           buckets = 65536
  if buckets < 1024:          buckets = 1024
  max_factor = 1                        # because hashsize was NOT specified
  nf_conntrack_max = max_factor × buckets
```

So **on any machine with more than 4 GiB of RAM, the kernel default is 262 144 buckets and `nf_conntrack_max = 262 144`.** If you *do* pass `hashsize` as a module parameter, `max_factor` is 8 instead of 1 — a compatibility carry-over — so `nf_conntrack_max` becomes `8 × hashsize`. This is the source of a lot of confusing folklore ("max is 8× buckets", "max is equal to buckets"); both are true, under different conditions.

Confirm on your host:

```bash
$ cat /proc/sys/net/netfilter/nf_conntrack_buckets \
      /proc/sys/net/netfilter/nf_conntrack_max \
      /sys/module/nf_conntrack/parameters/hashsize
262144
262144
262144
$ head -1 /proc/meminfo
MemTotal:       16461004 kB
```

*(Captured on Linux 6.18.5, 16 GB RAM.)* 16 GB > 4 GiB, so buckets took the 262 144 branch, and `max` equals it.

**Memory cost, computed properly.** Two allocations:

```
  entries:  nf_conntrack_max × objsize
            1 048 576 × 256 B  =  268 MB      ← non-swappable slab

  hash:     nf_conntrack_buckets × sizeof(hlist_nulls_head)
            262 144 × 8 B      =  2 MiB       ← does NOT grow when you raise max

  chain length at saturation = 2 × max / buckets      (two tuples per entry)
                             = 2 × 1 048 576 / 262 144 = 8

  Raising `max` alone therefore trades RAM for LOOKUP TIME. The lookup is a hash
  plus a linear walk of the bucket's chain; at chain length 8 you are doing up to
  8 tuple comparisons per packet instead of 2. Raise `hashsize` alongside `max`:

  $ echo 262144 > /sys/module/nf_conntrack/parameters/hashsize   # triggers a rehash
```

There is a hard stop on chain length in the kernel: confirmation refuses to insert into a chain longer than a randomised threshold between `MIN_CHAINLEN` (50) and `MIN_CHAINLEN + MAX_CHAINLEN` (80), incrementing `chaintoolong` and `insert_failed` and dropping the packet. If you ever see `chaintoolong` climbing, your bucket count is catastrophically undersized relative to `max`, or you are being hash-flooded.

**What happens at the ceiling.** This is more interesting than "the packet is dropped". From `__nf_conntrack_alloc()`:

```
  1. count = atomic_inc_return(&cnet->count)         # per-network-namespace count
  2. if count > nf_conntrack_max:
  3.     early_drop(net, hash):
  4.         for up to NF_CT_EVICTION_RANGE = 8 consecutive buckets:
  5.             walk the chain looking for a victim that is
  6.               · expired          → collect it, keep looking
  7.               · NOT IPS_ASSURED  → kill it, count early_drop++, RETURN SUCCESS
  8.             (ASSURED entries and entries from other netns are skipped)
  9.     if early_drop failed:
 10.         set conntrack_gc_work.early_drop = true      # arms the aggressive GC
 11.         atomic_dec(&cnet->count)
 12.         net_warn_ratelimited("nf_conntrack: table full, dropping packet")
 13.         return -ENOMEM    →  resolve_normal_ct fails  →  drop++  →  NF_DROP
```

Three things to take from that listing:

- **The table full path drops the packet and increments `drop`. It does not increment `insert_failed`.** Those two counters answer different questions (§6).
- **Half-open connections are sacrificed first.** `early_drop` only kills non-`ASSURED` entries, so under pressure the kernel preferentially evicts `SYN_SENT`/`SYN_RECV` state. That is a sensible policy and it produces a specific symptom: *established traffic is untouched while new connection attempts fail*, which is why the incident presents as "old connections fine, new ones hang".
- **Once the ceiling is hit, the GC changes gear.** `gc_worker` normally runs on an adaptive schedule between `GC_SCAN_INTERVAL_MIN` (1 s) and `GC_SCAN_INTERVAL_MAX` (60 s), bounded to `GC_SCAN_MAX_DURATION` (10 ms) per run, expiring timed-out entries. With `early_drop` armed it additionally evicts any non-assured, early-droppable entry while the count is above `nf_conntrack_max95` (95% of max). So the table tends to *stick* just below 95% during an incident rather than sitting pinned at 100% — do not expect `count == max` on the dashboard.

Note also that `cnet->count` is **per network namespace** while `nf_conntrack_max` is a global limit compared against it. Every pod netns therefore gets its own count checked against the same ceiling. In practice the host netns holds the SNAT entries for all pod egress, so it is the one that fills — but this is why a container can see `nf_conntrack_count = 0` while the node is saturated. Read the count in the host namespace.

### 6. The counters, and which incident each one means

`conntrack -S` prints one line per CPU, sourced from ctnetlink (`IPCTNL_MSG_CT_GET_STATS_CPU`), with a fallback to `/proc/net/stat/nf_conntrack`. The attributes the kernel actually sends are fixed in `ctnetlink_ct_stat_cpu_fill_info()`:

```
$ conntrack -S
cpu=0   found=41288 invalid=112 insert=0 insert_failed=0 drop=0 early_drop=0 \
        error=0 search_restart=907 clash_resolve=3 chaintoolong=0
cpu=1   found=39114 invalid=98  insert=0 insert_failed=0 drop=0 early_drop=0 \
        error=0 search_restart=884 clash_resolve=1 chaintoolong=0
...
```

*(Representative; the field names and their order are exact.)*

| Counter | Incremented when | What it means for you |
|---|---|---|
| `found` | a lookup hit an existing entry | normal traffic volume |
| `invalid` | a packet could not be associated with a valid flow | out-of-window TCP, late RSTs, asymmetric routing |
| `insert` | an entry was inserted at confirm | on modern kernels most inserts are counted elsewhere; often 0 |
| `insert_failed` | confirmation lost a race, or the chain was too long | **the DNS/UDP race, or hash undersizing** |
| `drop` | a packet was dropped by conntrack | **table full, or an unresolvable clash** |
| `early_drop` | an entry was evicted to make room | **you are at the ceiling and surviving on eviction** |
| `error` | an ICMP/ICMPv6 error could not be processed | usually noise |
| `search_restart` | a lockless lookup restarted due to a concurrent resize/move | noise unless enormous |
| `clash_resolve` | two CPUs raced to insert the same tuple and it was resolved | benign; the UDP race, *survived* |
| `chaintoolong` | insertion refused because the bucket chain exceeded ~50–80 | **buckets badly undersized** |

Three columns in `/proc/net/stat/nf_conntrack` — `new`, `ignore`, `delete` — are printed as literal zeros by the kernel (`ct_cpu_seq_show()` passes `0` for those positions); the corresponding ctnetlink attributes are marked "no longer used" in the uAPI header. Do not build an alert on them.

**The two incidents you must be able to tell apart:**

**Incident A — table exhaustion.** `drop` climbing, `early_drop` climbing, `nf_conntrack_count` near `nf_conntrack_max`, `nf_conntrack: table full, dropping packet` in `dmesg` (rate-limited, so the count in the log understates the drops by orders of magnitude). Symptom: new connections time out; established ones are fine. Fix: fewer flows, shorter timeouts, bigger table, different datapath.

**Incident B — the insertion race.** `insert_failed` climbing with `drop` climbing alongside it, `nf_conntrack_count` nowhere near the ceiling. This is the famous Kubernetes DNS problem. The mechanism, which you can now read directly out of §2's lifecycle:

- A pod's resolver sends the A and AAAA queries **in parallel from the same socket**, so two UDP packets with identical source tuples leave at nearly the same instant.
- They are processed on two different CPUs. Both take the `MISS` path at −200 and both allocate an `UNCONFIRMED` entry — legal, because unconfirmed entries are not in the hash and cannot see each other.
- DNAT at −100 translates both to the same CoreDNS backend, so they now have *identical* tuples.
- At `INT_MAX`, both call `__nf_conntrack_confirm()`. One wins. The other finds a clashing tuple and calls `nf_ct_resolve_clash()`, which for protocols whose tracker sets `allow_clash` (UDP does) may attach the loser's packet to the winner's entry and increment `clash_resolve`. When that is not possible, it increments **both `drop` and `insert_failed`** and returns `NF_DROP`.
- The dropped packet is a DNS query. glibc's resolver retries after `RES_TIMEOUT` = **5 seconds**, with `RES_DFLRETRY` = **2** attempts (`resolv/resolv.h`). Hence the signature symptom: DNS lookups that intermittently take exactly 5 s.

This was tracked as kubernetes/kubernetes#56903 ("DNS intermittent delays of 5s") and weaveworks/weave#3287; kernel-side clash-resolution improvements landed in the 5.x series, and the standard platform mitigations are NodeLocal DNSCache (a per-node caching agent, so the query never leaves the node and never needs NAT), `options single-request-reopen` in the pod's `resolv.conf` (use a separate socket per query, so the tuples differ), or using TCP for DNS.

**The metric to alert on.** In Prometheus terms, `node_exporter`'s `conntrack` collector exposes `node_nf_conntrack_entries` and `node_nf_conntrack_entries_limit` (and, on newer versions, `node_nf_conntrack_stat_drop`, `node_nf_conntrack_stat_insert_failed`, `node_nf_conntrack_stat_search_restart`). A serviceable pair of rules:

```promql
# Utilization — page before it bites.
100 * node_nf_conntrack_entries / node_nf_conntrack_entries_limit > 80

# The event itself — this is already an outage for someone.
rate(node_nf_conntrack_stat_drop[5m]) > 0
```

Both matter: the ratio is your lead indicator and it is *not* reliable on its own, because §5's 95% GC behaviour holds the ratio down while `drop` climbs. The `drop` rate is the ground truth.

### 7. kube-proxy on this path: iptables, IPVS, nftables

kube-proxy's job is to make a virtual ClusterIP work: a packet addressed to an IP that belongs to no interface must be rewritten to one of the Service's endpoint pods. That is DNAT, so it happens in the `nat` table at −100 (`PRE_ROUTING`) or on `LOCAL_OUT`, and — by §3 — every serviced flow gets a conntrack entry.

**kube-proxy also configures conntrack itself.** This is the single most important operational fact in this section and it is missed constantly. On startup and on every config sync, kube-proxy writes these sysctls (`pkg/proxy/conntrack/sysctls.go`, defaults in `pkg/proxy/apis/config/v1alpha1/defaults.go`):

| KubeProxyConfiguration field | Flag | Default | sysctl written |
|---|---|---|---|
| `conntrack.maxPerCore` | `--conntrack-max-per-core` | **32768** | `nf_conntrack_max` = 32768 × node CPUs |
| `conntrack.min` | `--conntrack-min` | **131072** | floor for the above |
| `conntrack.tcpEstablishedTimeout` | `--conntrack-tcp-timeout-established` | **24 h** | `nf_conntrack_tcp_timeout_established` |
| `conntrack.tcpCloseWaitTimeout` | `--conntrack-tcp-timeout-close-wait` | **1 h** | `nf_conntrack_tcp_timeout_close_wait` |
| `conntrack.tcpBeLiberal` | `--conntrack-tcp-be-liberal` | false | `nf_conntrack_tcp_be_liberal` |
| `conntrack.udpTimeout` | `--conntrack-udp-timeout` | 0 (leave alone) | `nf_conntrack_udp_timeout` |
| `conntrack.udpStreamTimeout` | `--conntrack-udp-stream-timeout` | 0 (leave alone) | `nf_conntrack_udp_timeout_stream` |

So on a Kubernetes node the effective ceiling is `max(32768 × numCPU, 131072)`, and the CPU count is read from sysfs rather than `runtime.NumCPU()` precisely so that a kube-proxy pinned by the CPU manager does not undercount the node. On current master a **cap of 1 048 576** was added to that scaled value; through **v1.34** the computation is uncapped (verified by reading `getConntrackMax` in both `pkg/proxy/conntrack/sysctls.go` on master and `cmd/kube-proxy/app/server_linux.go` on `release-1.34`). Concretely:

| Node | `numCPU` | `nf_conntrack_max` (≤ v1.34) | on master (capped) |
|---|---|---|---|
| small control-plane node | 2 | 131 072 (floor wins) | 131 072 |
| 16-core general worker | 16 | 524 288 | 524 288 |
| 8×A100 node | 96 | 3 145 728 | 1 048 576 |
| 8×H100 DGX-class node | 224 | 7 340 032 | 1 048 576 |

**Two corollaries worth internalising.** First, "the conntrack table is 262 144" is a *kernel* default and is usually wrong on a Kubernetes node; check `nf_conntrack_max` rather than assuming. Second, kube-proxy does not touch `nf_conntrack_buckets`, so a 224-core node running v1.34 gets `max = 7 340 032` against `buckets = 262 144` — an average chain length at saturation of 2 × 7 340 032 / 262 144 = **56**, uncomfortably close to the 50–80 `chaintoolong` refusal threshold. If you raise `max`, raise `hashsize` too.

Note also `tcpCloseWaitTimeout = 1 hour`, which *raises* the kernel's 60 s default. The reason is documented in the source and is worth knowing before you "optimise" it back down: if the `CLOSE_WAIT` entry expires while the local socket is still open, the eventual FIN from the local side is no longer SNATed correctly, never reaches the peer, and a later connection reusing the same `(source, port)` pair gets an RST — surfacing as sporadic `ECONNREFUSED` from `connect(2)` (kubernetes/kubernetes#32551, cited in the defaults source).

**iptables mode: the chain walk.** kube-proxy builds a chain hierarchy. `KUBE-SERVICES` is jumped to from `nat` `PREROUTING` and `OUTPUT`; it contains **one rule per Service port**, each matching destination IP + protocol + port and jumping to that Service's `KUBE-SVC-<hash>` chain. That chain selects an endpoint with probabilistic rules and jumps to a `KUBE-SEP-<hash>` chain that does the DNAT. Endpoint selection is:

```
-A KUBE-SVC-XPGD46QRK7WJZT7O -m statistic --mode random --probability 0.33333333349 \
   -j KUBE-SEP-AAAA...
-A KUBE-SVC-XPGD46QRK7WJZT7O -m statistic --mode random --probability 0.50000000000 \
   -j KUBE-SEP-BBBB...
-A KUBE-SVC-XPGD46QRK7WJZT7O -j KUBE-SEP-CCCC...
```

The probabilities are `1/n`, `1/(n−1)`, …, with the last rule an unconditional match — a sequential Bernoulli chain that yields a uniform 1/n choice overall. Each `KUBE-SEP-` chain is two rules: mark-for-masquerade if the source is the endpoint itself (hairpin), then `-j DNAT --to-destination 10.244.1.5:8080`.

**IPVS mode: the hash lookup.** kube-proxy in IPVS mode still uses iptables — but for *matching sets*, not for per-service dispatch. It maintains ipsets (`KUBE-CLUSTER-IP`, `KUBE-LOAD-BALANCER`, `KUBE-NODE-PORT-TCP`, `KUBE-LOOP-BACK`, …) and a handful of rules that match against them, then lets the in-kernel IPVS director do the actual load balancing against a virtual server table with the default scheduler `rr` (round robin). It also sets, on startup: `net/ipv4/vs/conntrack=1`, `net/ipv4/vs/expire_nodest_conn=1`, `net/ipv4/vs/expire_quiescent_template=1`, `net/ipv4/ip_forward=1` (all verified in `pkg/proxy/ipvs/proxier.go`).

**`net/ipv4/vs/conntrack=1` is the fact that kills the common misconception.** IPVS mode does *not* remove conntrack from the path — kube-proxy explicitly turns conntrack integration on, because masquerading still needs it. IPVS changes the **rule-evaluation cost**, not the **state cost**.

```
   THE SAME PACKET, TWO DATAPATHS — 5 000 Services × 5 endpoints each
   pod 10.244.3.7:51042  →  ClusterIP 10.96.140.22:80

 ┌────────────────────────── kube-proxy, iptables mode ─────────────────────────┐
 │                                                                              │
 │  nat PREROUTING ──▶ KUBE-SERVICES                                            │
 │                     ├─ rule    1: -d 10.96.0.10  --dport 53  -j KUBE-SVC-…   │
 │                     ├─ rule    2: -d 10.96.0.1   --dport 443 -j KUBE-SVC-…   │
 │                     ├─ rule    3: …                                          │
 │                     │      ⋮   ← EVERY rule is evaluated in order until      │
 │                     │      ⋮     one matches. This is a LINEAR WALK.         │
 │                     ├─ rule 2841: -d 10.96.140.22 --dport 80 -j KUBE-SVC-X   │ ✓
 │                     └─ … 2 159 more rules never reached for THIS packet      │
 │                                    │                                         │
 │                                    ▼                                         │
 │                            KUBE-SVC-X                                        │
 │                             ├─ statistic random p=0.20 → KUBE-SEP-A          │
 │                             ├─ statistic random p=0.25 → KUBE-SEP-B          │
 │                             ├─ statistic random p=0.33 → KUBE-SEP-C          │ ✓
 │                             ├─ statistic random p=0.50 → KUBE-SEP-D          │
 │                             └─ (unconditional)         → KUBE-SEP-E          │
 │                                    │                                         │
 │                                    ▼                                         │
 │                            KUBE-SEP-C : -j DNAT --to 10.244.7.19:8080        │
 │                                                                              │
 │  RULE SET SIZE   ≈ 5 000 svc + 25 000 endpoints ≈ tens of thousands of rules │
 │  MATCH COST      O(number of Services) — mean ~2 500 rule evaluations here   │
 │  PAID            once per FLOW (only ctinfo==NEW consults the nat table)     │
 │  CONTROL PLANE   iptables-restore rewrites the whole table; sync time grows  │
 │                  with ruleset size — the KEP-3866 "unfixable" problem        │
 │  CONNTRACK       1 entry per flow                                            │
 └──────────────────────────────────────────────────────────────────────────────┘

 ┌──────────────────────────── kube-proxy, IPVS mode ───────────────────────────┐
 │                                                                              │
 │  nat PREROUTING ──▶ KUBE-SERVICES                                            │
 │                     └─ -m set --match-set KUBE-CLUSTER-IP dst,dst            │
 │                        -j KUBE-MARK-MASQ                                     │
 │                        └── ipset = kernel hash lookup, O(1),                 │
 │                            ONE rule regardless of Service count              │
 │                                    │                                         │
 │                                    ▼                                         │
 │                       ┌────────────────────────────────────┐                 │
 │                       │  IPVS director (ip_vs)             │                 │
 │                       │  hash on (proto, vaddr, vport) ────┼──▶ O(1)         │
 │                       │                                    │                 │
 │                       │  virtual server 10.96.140.22:80 rr │                 │
 │                       │    ├─ real server 10.244.7.19:8080 │ ✓ scheduler     │
 │                       │    ├─ real server 10.244.2.31:8080 │   picks         │
 │                       │    ├─ real server 10.244.9.4:8080  │                 │
 │                       │    ├─ real server 10.244.1.88:8080 │                 │
 │                       │    └─ real server 10.244.5.7:8080  │                 │
 │                       │                                    │                 │
 │                       │  ip_vs_conn table  ← IPVS's OWN    │                 │
 │                       │  per-connection state (ipvsadm -Lnc)│                │
 │                       └────────────────────────────────────┘                 │
 │                                                                              │
 │  RULE SET SIZE   a few dozen rules + ipsets sized by Service count           │
 │  MATCH COST      O(1) hash — INDEPENDENT of Service count                    │
 │  CONTROL PLANE   incremental: add/remove one virtual or real server          │
 │  CONNTRACK       STILL 1 entry per flow — kube-proxy sets vs/conntrack=1     │
 │                  PLUS an ip_vs_conn entry. IPVS fixes LOOKUP, not STATE.     │
 └──────────────────────────────────────────────────────────────────────────────┘

  THE POINT: the two boxes differ in the middle column (how the backend is chosen)
  and are IDENTICAL in the bottom line (a conntrack entry per flow). Choosing IPVS
  to "fix conntrack" is a category error; choose it to fix rule-walk cost and
  control-plane sync time.
```

**nftables mode.** kube-proxy's nftables backend (KEP-3866, `NFTablesProxyMode`) went **alpha in v1.29, beta in v1.31, and stable in v1.33**. Its central data-plane change is replacing the linear `KUBE-SERVICES` walk with an nftables **verdict map** keyed on `ip daddr . ip protocol . th dport`, so dispatch is a single rule and a map lookup:

```
nft add rule ip kube-proxy services ip daddr . ip protocol . th dport vmap @service_ips
```

The KEP states the goal plainly: service dispatch becomes "roughly **O(1)** rather than **O(n)** as in the `iptables` backend." Its motivation section is also the clearest available statement of why the iptables backend cannot be fixed incrementally: the iptables API has no incremental update, so adding one rule means downloading the whole ruleset, editing it, and re-uploading it under a lock, and on the data-plane side the rule count is proportional to the Service count with every packet traversing all of it. Red Hat's deprecation of iptables in RHEL 9 (and likely removal in RHEL 10) is the other driver.

nftables mode does not change the conntrack story either: it is still DNAT, still one entry per flow.

### 8. Escaping the table (and what "escape" actually costs)

Four escapes exist, in increasing order of how much they change:

**(a) `--notrack` for flows that need no state.** In the `raw` table at −300, before conntrack at −200:

```bash
# Do not track NCCL's TCP bootstrap/health traffic on the storage VLAN at all.
iptables -t raw -A PREROUTING  -p tcp -s 10.10.0.0/16 -d 10.10.0.0/16 -j CT --notrack
iptables -t raw -A OUTPUT      -p tcp -s 10.10.0.0/16 -d 10.10.0.0/16 -j CT --notrack
```

Cost: those flows can no longer be NATed and cannot be matched by `-m conntrack --ctstate ESTABLISHED,RELATED`, so any stateful firewall rule covering them must be rewritten as stateless. Use it for high-volume, directly routed, intra-cluster traffic. It is surgical, and it is the only mechanism that removes entries rather than just making room for them.

**(b) Fewer flows.** HTTP keep-alive and a connection pool in the dataloader; HTTP/2 or gRPC multiplexing many requests over one connection; NodeLocal DNSCache so DNS never leaves the node. This is almost always the highest-leverage change and it is an application change, which is why platform teams reach for sysctls instead.

**(c) A bigger, better-shaped table.** Raise `nf_conntrack_max` *and* `hashsize` together, and shorten `TIME_WAIT` within the correctness limits of §4. Buys headroom; changes nothing structural.

**(d) A different datapath.** Cilium's eBPF datapath attacks the problem at three points:

- **Socket-level load balancing.** For traffic originating on the node (pod → ClusterIP), a BPF program attached to the cgroup `connect(2)`/`sendmsg` hooks rewrites the destination to a backend pod address **at connect time, before a packet exists**. There is no per-packet DNAT in the `nat` table, so no netfilter conntrack entry is needed for the Service translation — the socket is connected directly to the backend.
- **BPF host routing and DSR.** For pod-to-pod and north-south traffic, redirection happens at tc/XDP, bypassing the upper netfilter path; with Direct Server Return the load-balancing node keeps no return-path SNAT state.
- **Its own conntrack, when state is genuinely needed.** Stateful network policy and eBPF masquerade (`bpf.masquerade=true`) still require connection state; Cilium keeps it in BPF hash maps it owns (pinned under `/sys/fs/bpf/tc/globals/`, inspected with `cilium bpf ct list global`), sized and aged on its own terms rather than against `nf_conntrack_max`.

**This is not magic and it is not free.** The state does not disappear — it moves into a purpose-built, node-owned map with its own sizing, its own eviction, and its own metric to alert on. What you gain is that the map is not shared with everything else on the node in the same undifferentiated way, that translation for node-local traffic happens before packets exist, and that identity is available in the datapath for observability. [Lesson 08](08-ebpf.md) explains what a BPF map, a cgroup socket hook, and the verifier actually are; here you only need to know what they replace.

### 9. The GPU-fleet case: an inference node that fills its own table

Take a concrete node and do the arithmetic all the way through, because "conntrack can fill up" is not actionable and "you have 41 minutes of headroom at this request rate" is.

**The node.** 8 GPUs, 96 vCPUs, running a model-serving deployment behind a Service. Kubernetes v1.33, kube-proxy in iptables mode. Requests arrive from a gateway outside the cluster via a NodePort, and each replica fans out to a feature store and a vector database, both external, over HTTPS. The client library does not pool connections.

**Step 1 — establish the ceiling.** kube-proxy sets `nf_conntrack_max = max(32768 × 96, 131072) = 3 145 728` (v1.33, uncapped). Buckets remain at the kernel default 262 144.

```
  ceiling  M = 3 145 728 entries
  memory   = 3 145 728 × 256 B = 805 MB of non-swappable slab if fully occupied
  chain length at saturation = 2 × 3 145 728 / 262 144 = 24 comparisons per lookup
```

Already two findings before any traffic: 805 MB of a GPU node's RAM is a real budget line next to hugepages and pinned host buffers ([lesson 06](06-hugepages-thp-numa.md)), and a saturated chain length of 24 is a per-packet cost nobody accounted for. **Raise `hashsize` to 1 048 576 (8 MiB of buckets) and the chain length at saturation drops to 6.**

**Step 2 — count the flows per request.** One inbound request produces:

| Flow | Entries | Lifetime |
|---|---|---|
| gateway → NodePort (DNATed to a local pod) | 1 | duration + 120 s TIME_WAIT |
| pod → feature store (HTTPS, SNATed) | 1 | duration + 120 s |
| pod → vector DB (HTTPS, SNATed) | 1 | duration + 120 s |
| pod → DNS for each of the two hostnames (UDP, cached 30 s TTL) | ~0.07 | 30 s |

Call it **3 entries per request** with a ~120 s residency, ignoring DNS.

**Step 3 — find the saturating request rate.**

```
    N   =  λ_req × entries_per_request × (d + T)

    with d = 0.12 s (mean request duration), T = 120 s, 3 entries/request:

    N   =  λ_req × 3 × 120.12  =  360.4 × λ_req

    Saturation at N = M:
    λ_req = 3 145 728 / 360.4  ≈  8 728 requests/second
```

Roughly **8 700 req/s** before this node stops accepting new connections. That sounds comfortable — for an 8-GPU inference node it may well be more than the GPUs can serve. Now change one assumption at a time and watch it collapse:

```
  (i)  someone reverts hashsize to the default and raises nothing:
       still 8 728 req/s, but with chain length 24 the per-packet lookup cost
       roughly triples. You lose throughput before you lose the table.

  (ii) the node runs a kube-proxy older than the conntrack sysctl support,
       or kube-proxy is replaced by a CNI that does not set these sysctls,
       so the KERNEL default applies:  M = 262 144
       λ_req = 262 144 / 360.4  ≈  727 requests/second
       ── an 8-GPU inference node saturates conntrack at 727 req/s. ──

 (iii) the client adds a retry with a 1 s timeout and 3 attempts. Failed
       attempts sit in SYN_SENT for 120 s each, so each failed request now
       costs 3 entries at 120 s instead of 1. Once drops begin, occupancy
       per request TRIPLES, which causes more drops. This is the positive
       feedback loop that turns a slow degradation into a cliff.

 (iv)  the app switches to a pooled HTTPS client with keep-alive: the two
       egress flows become long-lived instead of per-request. At 64 pooled
       connections per replica and 8 replicas, egress occupancy drops from
       λ_req × 2 × 120 (=2.1 M entries at 8 700 req/s) to 512 entries.
       ── a 4 000× reduction from one client-library change. ──
```

**Step 4 — how fast does it fill?** Occupancy approaches steady state exponentially with time constant ≈ T, but the useful number during an incident is simpler. If the table is at `N₀` and the arrival rate jumps to λ with residency T, entries accumulate at `λ × entries_per_request` per second and drain at `N/T`:

```
    dN/dt = λ_e − N/T          where λ_e = λ_req × entries_per_request

    Time from N₀ to the ceiling M:
      t = −T · ln[ (λ_e·T − M) / (λ_e·T − N₀) ]

    Example: kernel-default table (M = 262 144), currently N₀ = 40 000,
    a traffic spike to λ_req = 1 500 req/s → λ_e = 4 500 entries/s, T = 120 s:

      λ_e·T = 540 000        (the steady state it is heading for — above M)
      t = −120 · ln[ (540 000 − 262 144) / (540 000 − 40 000) ]
        = −120 · ln(0.5557)
        = −120 × (−0.5876)
        ≈ 70 seconds

    ── Seventy seconds from a normal-looking table to hard drops. That is
       shorter than a 5-minute alert window, which is why you alert on the
       ratio at 80%, not on the drops. ──
```

**Step 5 — what the incident looks like.** NCCL is healthy, because RDMA never touches the host stack. GPU utilization is *high*, because in-flight requests complete normally. PSI is clean on every domain. The NIC reports zero drops. What you see is p99 latency exploding while p50 is unchanged (the p99 is the requests whose new connections were dropped and are waiting out a 5 s or 21 s connect timeout), a rising error rate on exactly the calls that open new connections, and — if you look — `drop` climbing in `conntrack -S`. **The measurement that resolves it in thirty seconds is `conntrack -C` against `sysctl net.netfilter.nf_conntrack_max`.**

## Perspectives

**Kernel-mechanism view.** The whole chapter reduces to one coupling: NAT cannot exist without conntrack, because the reverse translation has to live somewhere, and that somewhere is the reply tuple of an entry created at −200 and inserted at `INT_MAX`. Everything else — one entry per flow, `TIME_WAIT` residency, eviction policy, the confirmation race — is a consequence of that single structural fact plus the decision to make insertion the last thing that happens. The design is a classic kernel trade: pay once per flow to make every subsequent packet cheap, and accept that the "once per flow" is *state*, which is the resource that runs out.

**Operator/SRE view.** The symptom sequence is the on-call pattern-match, and it is worth being able to recite: *new connections hang or time out while established ones are fine, clustered on your busiest nodes, with no NIC drops and clean PSI.* Most engineers reach for "check the network," which spends the first twenty minutes of the incident at the wrong layer. The experienced move is `conntrack -C` and `conntrack -S` first, precisely because the symptom is so easy to misattribute. And the second-order discipline: know which counter you are reading. `drop` climbing with a near-full table is exhaustion; `insert_failed` climbing with an empty table is the UDP race and has a completely different fix.

**GPU-fleet-specific view.** This is the headline case for the module because the two traffic classes on a GPU node have *opposite* exposure. Gradient traffic over RDMA is genuinely immune — it bypasses the host stack. Everything that funds the job — checkpoint upload, dataset prefetch, control-plane calls, the inference request path — is ordinary TCP through netfilter. The result is a confusing incident shape: **"NCCL is healthy, but checkpoint uploads intermittently time out."** Recognising that RDMA bypass and conntrack exhaustion are orthogonal, rather than concluding that a healthy fabric rules out a network-layer problem, is the specific expertise this role pays for. The economics sharpen it: a 70-second fill time (§9) against a job whose restart costs an hour of an 8-GPU node means the difference between a dashboard alert and a five-figure loss is a threshold on a ratio nobody had graphed.

**Economics and architecture view.** Step back and the conntrack-versus-eBPF choice is a **state-ownership and blast-radius decision**, not a scaling knob. A node-global table means every pod on that node — regardless of tenant, team, or workload class — shares one finite resource with no per-pod quota and no priority. One broken retry loop, one unbounded crawler, one misconfigured sidecar starves every other tenant's NATed egress on that node, and nothing about the victims' traffic changed. That is a *fairness* failure, not a capacity failure, and the distinction determines the fix: raising the ceiling buys time; per-workload accounting requires a datapath that can attribute state to an identity. This is the framing a staff-level design review actually probes — not "does raising `nf_conntrack_max` close the incident" but "should one tenant be able to do this to its neighbours at all, and what does the alternative cost us in operational surface?"

## Real-world use cases

- **kubernetes/kubernetes#56903 — "DNS intermittent delays of 5s."** *What happened:* pods saw DNS resolution intermittently take exactly five seconds. *The mechanism:* glibc's resolver sends A and AAAA queries in parallel from one socket; both UDP packets are processed concurrently on different CPUs, both create unconfirmed conntrack entries, both get DNATed to the same CoreDNS backend, and at confirmation one loses the race and is dropped, incrementing `insert_failed` (and `drop`). *Why exactly five seconds:* glibc's `RES_TIMEOUT` is 5 s with `RES_DFLRETRY` = 2 (`resolv/resolv.h`). *The numbers to watch:* `insert_failed` in `conntrack -S`, with `nf_conntrack_count` nowhere near the ceiling. *The fixes:* NodeLocal DNSCache (query never leaves the node, so no NAT and no race), `options single-request-reopen` (a separate socket per query gives the two queries different tuples), TCP for DNS, and kernel-side clash-resolution improvements in the 5.x series. Tracked in parallel as weaveworks/weave#3287.

- **kubernetes/kubernetes#32551 — `CLOSE_WAIT` expiry causing sporadic `ECONNREFUSED`.** *What happened:* connections through kube-proxy's SNAT would occasionally fail to connect. *The mechanism, quoted in substance from the comment kube-proxy's own defaults carry:* `CLOSE_WAIT` is a half-close state that persists while the local socket stays open (a lazily garbage-collected socket, typically). If the conntrack entry expires at the kernel's 60 s default while the socket is still open, the eventual local FIN is no longer SNATed, never reaches the peer, and the peer's state diverges — so a new connection reusing the same source IP and port is met with an RST, which `connect(2)` reports as `ECONNREFUSED`. *The fix:* kube-proxy defaults `--conntrack-tcp-timeout-close-wait` to **one hour** to outlast typical server-side `FIN_WAIT` timers. *What it shows:* a timeout can be too *short* as well as too long, and conntrack timeouts must be reasoned about against the peer's state machine, not just against table occupancy.

- **KEP-3866, "Add an nftables-based kube-proxy backend"** (alpha v1.29, beta v1.31, stable v1.33). *What it argues:* iptables has two unfixable problems. On the control plane, the API supports no incremental change — adding one rule means downloading, editing, and re-uploading the entire ruleset under a lock, so sync time grows with the number of Services and `iptables-restore` parse time alone becomes substantial. On the data plane, rule count is proportional to Service count and every packet traverses all of it. *The fix:* an nftables verdict map keyed on destination address, protocol, and port, making dispatch "roughly O(1) rather than O(n)". *The other driver:* Red Hat deprecated iptables in RHEL 9 and is likely to remove it in RHEL 10, which would strand kube-proxy's `iptables`, `ipvs`, and `userspace` modes alike, all of which use the iptables API. *What it shows:* the O(n) chain walk in §7's diagram is not a straw man; it is the upstream project's own stated reason for building a replacement.

## Worked example

**The incident: "checkpoint uploads from training pods intermittently time out. NCCL is fine."**

*(Transcripts below are representative reconstructions with internally consistent arithmetic, not a captured session. The Practice section produces your own.)*

**Step 1 — triage the shape, not the layer.** NCCL over RDMA is healthy, which rules out the fabric but rules out *nothing* about the host stack — the two are orthogonal. The failing thing is TCP egress to object storage, and the failures are connect timeouts, not resets or 5xx. Silent SYN loss with healthy established connections is the conntrack fingerprint. Check it before you page netops.

**Step 2 — measure headroom, in the host namespace.**

```bash
$ sudo conntrack -C
2 981 442
$ sysctl net.netfilter.nf_conntrack_max net.netfilter.nf_conntrack_buckets
net.netfilter.nf_conntrack_max = 3145728
net.netfilter.nf_conntrack_buckets = 262144
```

2 981 442 / 3 145 728 = **94.8%**. Note that it is sitting just under 95% rather than pinned at 100% — that is §5's `gc_worker` behaviour, which arms aggressive eviction above `nf_conntrack_max95`. **A table that looks "not quite full" is exactly what a saturated table looks like.**

Also note `buckets` = 262 144 against `max` = 3 145 728: chain length at this occupancy is 2 × 2 981 442 / 262 144 ≈ **23**. Every packet on this node is paying a 23-deep chain walk.

**Step 3 — read the counters, and read them correctly.**

```bash
$ sudo conntrack -S | awk '{for(i=1;i<=NF;i++) if($i ~ /^(drop|insert_failed|early_drop|chaintoolong)=/) printf "%s ", $i; print ""}' \
  | awk -F'[= ]' '{d+=$2; f+=$4; e+=$6; c+=$8} END {print "drop="d, "insert_failed="f, "early_drop="e, "chaintoolong="c}'
drop=48213 insert_failed=0 early_drop=1904772 chaintoolong=0
$ dmesg | grep -c 'nf_conntrack: table full'
41
```

Read this precisely:

- `early_drop = 1 904 772` — the kernel has evicted nearly two million non-`ASSURED` entries to make room. **The node has been surviving on eviction for a long time.** Every one of those was somebody's half-open connection.
- `drop = 48 213` — the eviction scan failed 48 213 times and the packet was dropped. These are the checkpoint uploads.
- `insert_failed = 0` — this is *not* the UDP confirmation race. Different incident, don't chase DNS.
- `chaintoolong = 0` — chains are long (23) but under the 50–80 refusal threshold. Not yet fatal, but no headroom.
- The `dmesg` count of 41 badly understates the damage: `net_warn_ratelimited()` suppresses repeats, so 41 log lines correspond to 48 213 drops.

**Step 4 — attribute the occupancy.** A full `-L` dump on a 3 M-entry table is expensive; do it once, cheaply, and group:

```bash
$ sudo conntrack -L 2>/dev/null | awk '{print $5}' | sort | uniq -c | sort -rn | head -5
2 216 903 src=10.244.7.9
  204 118 src=10.244.7.11
   88 402 src=10.244.3.7
   61 240 src=10.244.0.19
   12 006 src=10.244.7.4
```

One pod holds 74% of the table. Confirm what it is doing:

```bash
$ sudo conntrack -L -s 10.244.7.9 2>/dev/null | awk '{print $4}' | sort | uniq -c | sort -rn
2 106 551 TIME_WAIT
  104 219 ESTABLISHED
    6 133 SYN_SENT
$ kubectl get pod -A -o wide --field-selector status.podIP=10.244.7.9
NAMESPACE   NAME                     READY   NODE
training    llm-pretrain-w3-loader   1/1     gpu-node-14
```

**95% of that pod's entries are `TIME_WAIT`.** The prefetcher is opening one HTTPS connection per object and closing it. Back out the rate from Little's law:

```
    N_TW = λ × T_TW
    2 106 551 = λ × 120
    λ ≈ 17 555 new connections per second
```

Seventeen thousand connections per second, from one pod, to fetch shards.

**Step 5 — mitigate, in the order that respects the trade-offs.**

```bash
# (a) Fix the shape of the table first — this is free and has no correctness risk.
$ echo 1048576 | sudo tee /sys/module/nf_conntrack/parameters/hashsize
# chain length at current occupancy: 2 × 2 981 442 / 1 048 576 ≈ 5.7  (was 23)

# (b) Buy headroom. Budget the RAM explicitly: 6 291 456 × 256 B ≈ 1.6 GB slab.
$ sudo sysctl -w net.netfilter.nf_conntrack_max=6291456

# (c) Shorten TIME_WAIT — datacentre-internal RTT is sub-millisecond, so 30 s is
#     still ~30 000× the RTT. Cuts steady-state occupancy 4×.
$ sudo sysctl -w net.netfilter.nf_conntrack_tcp_timeout_time_wait=30

# Persist, and do it where kube-proxy will not overwrite it:
$ cat | sudo tee /etc/sysctl.d/90-conntrack.conf <<'CONF'
net.netfilter.nf_conntrack_max = 6291456
net.netfilter.nf_conntrack_tcp_timeout_time_wait = 30
CONF
```

**Do not simply write these and walk away.** kube-proxy rewrites `nf_conntrack_max` on every sync from `--conntrack-max-per-core × numCPU`, so a sysctl drop-in will be silently reverted. The durable change is to raise `conntrack.maxPerCore` in the `KubeProxyConfiguration` ConfigMap (or set `--conntrack-max-per-core`) and restart the kube-proxy DaemonSet. Verify afterwards that the value stuck:

```bash
$ sysctl net.netfilter.nf_conntrack_max
net.netfilter.nf_conntrack_max = 6291456
```

**Step 6 — the actual fix.** Everything in step 5 is a stopgap. Two real fixes, in order of leverage:

1. **Application side.** Give the prefetcher an HTTP connection pool with keep-alive, or use a client that does (an S3 SDK with a configured `max_pool_connections`, `http.Transport` with `MaxIdleConnsPerHost`, an `aiohttp` session reused across fetches). Going from 17 555 conn/s to a pool of, say, 256 connections per worker × 8 workers takes egress occupancy from 2.1 M entries to about 2 048 — a factor of a thousand — and *also* removes 17 555 TLS handshakes per second, which was silently costing CPU that the GPUs were waiting on.
2. **Platform side.** For traffic that is directly routed and needs no NAT, `-j CT --notrack` in the `raw` table removes it from the table entirely. For Service traffic, move the node to an eBPF datapath so ClusterIP translation happens at `connect(2)` and never creates a netfilter entry; then monitor the BPF CT map instead (`cilium bpf ct list global | wc -l`) and set an alert on *that*.

**Root-cause statement for the postmortem.** Not "flaky network." *A node-global, per-flow NAT state table was exhausted by one pod's connection churn — 17 555 new connections per second, each holding a table slot for 120 seconds after closing — dropping new-connection SYNs for every tenant on the node while established traffic and RDMA were unaffected. Contributing factors: `nf_conntrack_buckets` left at the kernel default while `nf_conntrack_max` was scaled by kube-proxy to 12× it, giving a 23-deep chain walk per packet; and no alert on `nf_conntrack_count / nf_conntrack_max`.*

## Practice

**Environment:** a Linux VM or a privileged container you can break — **not** a shared or production node. You need root or `CAP_NET_ADMIN`, `conntrack-tools`, `iptables`, `curl`, and ideally `iperf3` or `nc`. Confirm `nf_conntrack` is loaded (`lsmod | grep nf_conntrack`; it autoloads when a rule that needs it is inserted).

**Feeds:** [Anatomy of a Container](../practice/anatomy-of-a-container/README.md) — commit the findings and the sysctl playbook into its diagnostic toolkit.

1. **Baseline and budget.** Record `nf_conntrack_count`, `nf_conntrack_max`, `nf_conntrack_buckets`, and the `objsize` for `nf_conntrack` from `/proc/slabinfo`. Compute, from *your* numbers: the RAM a full table would consume, and the average hash chain length at saturation. State whether your `max`/`buckets` ratio is sane.

2. **Watch an entry's whole life.** Start `conntrack -E` in one shell. In another, make a single outbound TCP connection (`curl -s https://example.com > /dev/null` or `nc -z <host> 443`). Capture the `[NEW]`, `[UPDATE]`, and `[DESTROY]` events, and note the state each `[UPDATE]` reports. Then immediately run `conntrack -L | grep <your port>` and record the state and the remaining timeout. Watch it sit in `TIME_WAIT` and confirm it survives ~120 s after the connection closed.

3. **Prove NAT lives in the reply tuple.** Create a NATed path — the simplest is a network namespace with a veth pair and a MASQUERADE rule in `POSTROUTING` — then make a connection through it and dump the entry. **Write down both tuples and point at the field that is the NAT mapping.** If you cannot build the namespace, reproduce it with a Docker container on the default bridge, which masquerades by default, and dump the entry from the host.

4. **Reproduce exhaustion deliberately.** On the disposable VM only: `sysctl -w net.netfilter.nf_conntrack_max=256`, then generate more than 256 concurrent flows (a loop of 500 backgrounded `curl` calls to a local server, or `nc` in a loop). Capture: the `nf_conntrack: table full, dropping packet` line from `dmesg`; the change in `drop` and `early_drop` from `conntrack -S` before and after; and evidence that an *already established* connection survived while new ones failed (start a long-lived `nc` or an `iperf3` stream first, and show it kept running). Restore the ceiling afterwards.

5. **Distinguish the two incidents.** From your step-4 capture, confirm `insert_failed` did **not** move while `drop` did. Then write two or three sentences explaining what a rising `insert_failed` with a nearly empty table would have meant instead, and what you would do about that one.

6. **Size a table from a rate.** Pick a workload you actually run (or the §9 inference node) and compute: entries per request, steady-state occupancy at your peak rate, the request rate at which your node's real `nf_conntrack_max` saturates, and the time-to-fill from a 40% baseline under a 2× traffic spike. Show the arithmetic with units.

7. **Take one flow out of the table.** Add `iptables -t raw -A OUTPUT -p tcp -d <some host> -j CT --notrack`, make a connection to that host, and show that no conntrack entry appears for it. Then explain in one sentence what you gave up.

**Acceptance:** a note that shows (1) baseline `count`/`max`/`buckets` plus your computed RAM budget and saturation chain length; (2) one annotated `conntrack -L` entry with the two tuples labelled and the NAT mapping identified; (3) the exhaustion evidence — the `table full` log line and/or non-zero `drop` — together with the explicit statement "established connections survived, new ones were dropped," backed by your long-lived stream; (4) the `drop`-versus-`insert_failed` distinction in your own words; and (5) the sizing arithmetic from step 6 with the two sysctls you would change and the trade-off each one carries.

## Common pitfalls

1. **Treating "intermittent connection failures" as a network-layer problem.** *Symptom:* new connections time out on your busiest nodes; the NIC reports zero drops; PSI is clean. *Mechanism:* the drop happened in software at `NF_INET_PRE_ROUTING`, in `__nf_conntrack_alloc()`, and produced no ICMP and no RST — so every instrument that watches the wire shows a healthy link. Check `conntrack -C` against `nf_conntrack_max` before escalating.

2. **Assuming `nf_conntrack_max` is 262 144 on a Kubernetes node.** *Symptom:* your capacity math is off by an order of magnitude in either direction. *Mechanism:* 262 144 is the *kernel* default for a machine with more than 4 GiB of RAM, but kube-proxy overwrites it with `max(32768 × numCPU, 131072)` on every sync. On a 96-core node that is 3 145 728. Read the live value; and if you set it by sysctl, expect kube-proxy to revert it.

3. **Raising `nf_conntrack_max` without raising `hashsize`.** *Symptom:* per-packet cost creeps up and, in the extreme, `chaintoolong` starts incrementing and insertions fail even though the table is not full. *Mechanism:* buckets do not scale with `max`; average chain length at saturation is `2 × max / buckets`, and confirmation refuses to insert into a chain longer than a randomised 50–80. Scale both, and remember that the hash table is only 8 bytes per bucket.

4. **Reading `drop` and `insert_failed` as the same signal.** *Symptom:* you chase table exhaustion during a DNS incident, or chase DNS during an exhaustion incident. *Mechanism:* table-full increments `drop` only (`resolve_normal_ct` fails, `NF_DROP`); a lost confirmation race increments *both*, and a chain-too-long refusal increments `insert_failed` and `chaintoolong`. `insert_failed` climbing with a mostly empty table is the UDP confirmation race, whose fix (NodeLocal DNSCache, `single-request-reopen`) has nothing to do with table size.

5. **Concluding the table is fine because `count` is below `max`.** *Symptom:* the ratio hovers at 94% and never reaches 100%, so nobody believes it is full. *Mechanism:* once an allocation fails, `gc_worker` arms `early_drop` and evicts non-`ASSURED` entries whenever the count exceeds 95% of `max`, so a saturated table sits just below the ceiling by construction. `early_drop` and `drop` are the ground truth; the ratio is only a lead indicator.

6. **Believing RDMA/NCCL health rules out a host-stack problem.** *Symptom:* "the network is fine, NCCL is at full bandwidth" while checkpoint uploads fail. *Mechanism:* RoCE/IB bypasses the host TCP/IP stack, so gradient traffic is genuinely immune to conntrack — and every other flow on the same node is not. Fabric health and netfilter health are orthogonal; testing one tells you nothing about the other.

7. **Thinking a different datapath means "no more connection tracking."** *Symptom:* you migrate to IPVS or Cilium to fix an exhaustion incident, and either the incident does not move or you delete the conntrack alert and go blind. *Mechanism:* kube-proxy's IPVS mode explicitly sets `net/ipv4/vs/conntrack=1` because masquerading still needs netfilter state, and adds its own `ip_vs_conn` table on top — it fixes the O(n) rule walk, not the state. Cilium's socket-LB does remove the *Service DNAT* entry (translation happens before a packet exists), but stateful policy and eBPF masquerade still need state, kept in Cilium's own BPF maps with their own sizing and eviction. The bottleneck moves and becomes attributable; it does not vanish. Alert on whatever now holds the state.

8. **Shortening `TIME_WAIT` (or any timeout) without reasoning about the peer.** *Symptom:* sporadic RSTs or `ECONNREFUSED` after a "harmless" tuning change. *Mechanism:* tuple reuse before delayed segments have drained lets an old segment match a new flow; and, as kubernetes#32551 showed for `CLOSE_WAIT`, expiring an entry while a socket is still open desynchronises your state from the peer's so the next connection reusing that source port is refused. Timeouts are a negotiation with the network's segment lifetime and with the peer's state machine, not a free dial.

9. **Flushing the table to "clear it."** *Symptom:* `conntrack -F` "fixes" the alert and immediately breaks every NATed connection on the node. *Mechanism:* the flush deletes the reply tuples, which *are* the NAT mappings; in-flight replies then arrive with no entry, cannot be reverse-translated, and are dropped or delivered to the wrong place. Use `conntrack -D` with a filter for surgery, never `-F` on a live node.

## Self-check

**(a) Trace a pod's outbound packet to an external HTTPS endpoint through netfilter, naming the hooks in order and saying where conntrack and NAT act.**
**Answer:** On the host the packet is *forwarded*, so it enters `NF_INET_PRE_ROUTING`, where hooks run in priority order: conntrack defrag (−400), the `raw` table (−300, the only place a rule can set `--notrack`), **conntrack (−200)**, which builds the tuple, hashes it, misses, and allocates an `UNCONFIRMED` `struct nf_conn`; `mangle` (−150); the `nat` table's DNAT direction (−100), consulted only because `ctinfo == NEW`. The routing decision then sends it to `NF_INET_FORWARD` (`filter` at 0), and on to `NF_INET_POST_ROUTING`: `mangle` (−150), the `nat` table's SNAT direction (100) where `MASQUERADE` rewrites **the reply tuple** and derives the packet edit from it, and finally `nf_confirm` at `INT_MAX`, which inserts both tuplehashes into the hash table. Locally generated packets enter at `NF_INET_LOCAL_OUT` instead (conntrack −200, nat DST −100, filter 0); packets terminating locally are confirmed at `NF_INET_LOCAL_IN` instead of `POST_ROUTING`.

**(b) Why does pod egress NAT create exactly one conntrack entry per flow, and why can that not be optimised away?**
**Answer:** Pod addresses are not routable outside the cluster, so `MASQUERADE` rewrites the source to the node address; for the reply to find the right pod and port the reverse mapping must be stored, and netfilter stores it in the entry's **reply tuple**, which after NAT is no longer the plain inversion of the original. `nf_nat_setup_info()` edits that tuple and the packet rewrite is derived from it, so the mapping and the entry are the same object — which is why `nat` rules are evaluated only on the first packet (`ctinfo == NEW`) and every later packet is translated by replaying the stored tuples. One flow needs one mapping, one mapping is one entry, and entries cannot be shared because tuples are unique by construction. The only ways to reduce the count: fewer flows (pooling, HTTP/2, NodeLocal DNSCache), shorter residency (timeouts), or flows that are neither NATed nor tracked (`-j CT --notrack`, direct routing, or a datapath that translates before packets exist).

**(c) Compute the steady-state conntrack occupancy of a service opening 1 200 short HTTPS connections per second with a mean duration of 80 ms, and the rate at which it would saturate a kernel-default table.**
**Answer:** Little's law: `N = λ × (d + T)`, with `T` = 120 s of `TIME_WAIT`. `N = 1 200 × 120.08 ≈ 144 096 entries` — of which the 80 ms of actual work contributes 96 and `TIME_WAIT` contributes 144 000, so the table is 99.93% dead flows. Inverting against the kernel default `nf_conntrack_max = 262 144` (the value on any machine over 4 GiB, since `buckets` takes the 262 144 branch and `max_factor` is 1): `λ_max ≈ 2 183 conn/s`. On a Kubernetes node the real ceiling is `max(32768 × numCPU, 131072)` because kube-proxy overwrites it — 3 145 728 on a 96-core node, so `λ_max ≈ 26 197 conn/s`, costing 805 MB of non-swappable slab at saturation and a chain length of `2 × max / buckets` = 24, which is the argument for raising `hashsize` alongside `max`.

**(d) What exactly produces `nf_conntrack: table full, dropping packet`, what happens just before it, and which counters move?**
**Answer:** In `__nf_conntrack_alloc()` the per-netns count is incremented and compared against the global `nf_conntrack_max`. If it exceeds it, the kernel first calls `early_drop()`, which scans up to `NF_CT_EVICTION_RANGE` = 8 consecutive buckets for a victim, reaping expired entries and killing the first **non-`IPS_ASSURED`** confirmed entry it finds (incrementing `early_drop`); the allocation then proceeds. Only if that scan finds nothing does the kernel arm the aggressive GC (`gc_worker` then evicts droppable entries whenever the count exceeds 95% of `max`), decrement the count, emit the rate-limited warning, and return `-ENOMEM` — which propagates through `resolve_normal_ct()`, increments **`drop`**, and drops the packet. So `early_drop` (usually far larger — the node has been surviving on eviction) and `drop` move; **`insert_failed` does not**, because it counts confirmation-time failures instead. `net_warn_ratelimited()` suppresses repeats, so the `dmesg` line count understates drops by orders of magnitude. And because eviction prefers non-assured entries, half-open connections die while established flows are untouched.

**(e) An engineer reports DNS lookups from pods that intermittently take exactly five seconds, on a node whose conntrack table is 12% full. What is happening and what do you check?**
**Answer:** This is the confirmation race, not exhaustion. glibc's resolver sends the A and AAAA queries in parallel from one socket; both packets miss the hash at −200 on different CPUs and both allocate `UNCONFIRMED` entries — legal, because unconfirmed entries are not in the hash and cannot see each other — and DNAT at −100 makes their tuples identical. At `INT_MAX` both confirm; one wins, the other calls `nf_ct_resolve_clash()`. UDP sets `allow_clash`, so the loser's packet can often be attached to the winner's entry (`clash_resolve`); when it cannot, the kernel increments **both `drop` and `insert_failed`** and drops it. glibc then retries after `RES_TIMEOUT` = 5 s (`RES_DFLRETRY` = 2) — the exact five seconds. **Check `insert_failed` in `conntrack -S`**: non-zero with a table nowhere near `nf_conntrack_max` is the signature. Fixes: NodeLocal DNSCache (queries never leave the node, so no NAT), `options single-request-reopen` (a fresh socket per query, so different source ports), TCP for DNS, and a 5.x-or-later kernel. Tracked as kubernetes/kubernetes#56903 and weaveworks/weave#3287.

**(f) Rank kube-proxy's iptables mode, IPVS mode, nftables mode, and a Cilium eBPF datapath by (i) per-packet lookup cost and (ii) conntrack entries per flow.**
**Answer:** *(i) Lookup cost.* iptables mode walks `KUBE-SERVICES` linearly — one rule per Service port — so match cost is O(Services), paid on the first packet of each flow; the control-plane cost is worse, because the iptables API has no incremental update and every sync rewrites the whole ruleset under a lock. IPVS mode replaces that with ipset matching (a kernel hash, one rule regardless of Service count) plus the IPVS director's hash on `(proto, vaddr, vport)`, default scheduler `rr`. nftables mode (KEP-3866: alpha v1.29, beta v1.31, **stable v1.33**) uses a verdict map keyed on `ip daddr . ip protocol . th dport` — "roughly O(1) rather than O(n)" — plus incremental updates. Cilium's socket-LB translates inside `connect(2)`, so there is no service dispatch on the packet path at all. *(ii) Entries per flow.* iptables, IPVS, and nftables modes are **identical**: one netfilter entry per flow, because all three implement Services as DNAT and DNAT stores its mapping in the entry. IPVS additionally adds an `ip_vs_conn` entry and kube-proxy sets `net/ipv4/vs/conntrack=1` at startup, so IPVS is *more* state, not less. Only the eBPF datapath changes this, and where state is still needed (stateful policy, `bpf.masquerade=true`) it lives in Cilium's own maps, sized independently of `nf_conntrack_max`. **Three of the four fix lookup cost and change nothing about state; only the fourth changes where state lives.**

**(g) Why is conntrack exhaustion better framed as a fairness failure than a scaling limit?**
**Answer:** A scaling-limit framing implies the fix is always "raise the ceiling." But `nf_conntrack_max` is a single, node-global, unpartitioned resource with no per-pod quota and no priority — `early_drop` picks victims by whether they are `ASSURED`, not by whose they are. One pod with a broken retry loop can therefore consume nearly the whole table and starve every other tenant, exactly as in the worked example where one loader pod held 74% of a 3 M-entry table and the visible casualty was a different workload's checkpoint upload. Raising the ceiling buys time proportional to the increase and changes nothing about isolation; the architectural fix is a datapath that can attribute state to a workload identity and bound it, which is what per-identity BPF maps enable ([lesson 08](08-ebpf.md)).

**(h) What is the advantage of `conntrack -E` over polling `conntrack -L`, and when would you still use `-L`?**
**Answer:** `conntrack -L` copies the **entire** confirmed table across netlink on every invocation, so a one-second poll loop on a million-entry table is itself a load generator that perturbs what you are measuring. `conntrack -E` subscribes to ctnetlink and streams `[NEW]`/`[UPDATE]`/`[DESTROY]` as they happen, giving lifecycle transitions at zero dump cost — the right tool for watching churn and catching which pod is creating flows. (It depends on `nf_conntrack_events`, default `2` = "auto": the event extension is allocated only when a listener exists.) Use `-L` for a one-shot census grouped by source, `conntrack -C` when you only need the count (one cheap round trip), and `-L unconfirmed` to see entries that exist on an `skb` but are not yet inserted.

## Connections & what's next

This lesson sits with [lesson 03](03-cgroups-v2-and-k8s-enforcement.md) (cgroups v2), [lesson 04](04-psi.md) (PSI), and [lesson 06](06-hugepages-thp-numa.md) (hugepages/NUMA) in the module's central theme: **fleet-scale failures come from shared, finite kernel resources that look fine in aggregate and fail per-tenant.** Conntrack is the sharpest instance because it has the least isolation — no per-cgroup accounting, no quota, no pressure metric. It is also the counter-example that keeps lesson 04 honest: PSI is the best saturation instrument you have and it cannot see this at all.

Carry three things forward. **First**, the structural fact: state lives where translation lives, so any translation mechanism you adopt inherits a state-sizing problem, whether the state is called `nf_conntrack`, `ip_vs_conn`, or `cilium_ct4_global`. **Second**, the counter discipline: `drop`, `insert_failed`, and `early_drop` are three different incidents, and knowing which is which is most of the diagnosis. **Third**, the sizing habit: entries = rate × lifetime, and both terms are things you can change.

Next: **[08 · eBPF — the observability substrate](08-ebpf.md)**, which explains the mechanism this lesson took on faith. Once you know what a BPF program attached to a cgroup socket hook actually *is* — what the verifier proves about it, what a map costs, where in the datapath it can attach — "Cilium removes conntrack load" stops being a vendor claim and becomes something you can verify yourself with `bpftool` and `cilium bpf ct list`. It is also the tool that lets you watch this lesson's mechanisms live: a kprobe on `__nf_conntrack_alloc` or a tracepoint on the netfilter hooks turns everything above into a trace with timestamps and stacks.

## References & further reading

> **A note on verification.** This environment's egress proxy blocks `kernel.org`, `docs.kernel.org`, `kubernetes.io`, `man7.org`, `git.netfilter.org`, and most vendor blog domains. Everything marked **[verified against source]** below was read directly from the upstream Git repositories via `raw.githubusercontent.com`. Entries marked **[not reachable]** are listed as optional further reading only; no claim in this lesson depends on them.

**Primary sources — kernel**

1. **`Documentation/networking/nf_conntrack-sysctl.rst`** (Linux, current master) — https://docs.kernel.org/networking/nf_conntrack-sysctl.html — **[verified against source]**, read from `torvalds/linux`. The authoritative list of every conntrack sysctl and its default: `nf_conntrack_buckets` (RAM/16384, clamped to 1024–262144), `nf_conntrack_max` (defaults to `nf_conntrack_buckets`), all TCP/UDP/ICMP/SCTP/GRE timeouts in §4, `nf_conntrack_events` (default 2 = auto), `nf_conntrack_acct`, `nf_conntrack_tcp_be_liberal`, and the note that each entry occupies the hash twice. **Correction recorded:** the document states `nf_conntrack_expect_max` defaults to `nf_conntrack_buckets / 256`; the source computes `nf_ct_expect_max = (nf_conntrack_htable_size / 256) × 4`, which matches the observed 4096 on a host with 262 144 buckets. Where doc and source disagree, this lesson follows the source.
2. **`net/netfilter/nf_conntrack_core.c`** — https://github.com/torvalds/linux/blob/master/net/netfilter/nf_conntrack_core.c — **[verified against source]**. Ground truth for: the bucket/`max_factor` computation in `nf_conntrack_init_start()`; the 256-byte `kmem_cache_create("nf_conntrack", …)`; `__nf_conntrack_alloc()`'s ceiling check and the exact `net_warn_ratelimited("nf_conntrack: table full, dropping packet")` string; `early_drop()` with `NF_CT_EVICTION_RANGE = 8` and its `IPS_ASSURED` exclusion; `gc_worker()` with `GC_SCAN_INTERVAL_MIN/MAX` (1 s / 60 s), `GC_SCAN_MAX_DURATION` (10 ms), and `nf_conntrack_max95`; `__nf_conntrack_confirm()` with `MIN_CHAINLEN = 50` / `MAX_CHAINLEN = 30`; and every `NF_CT_STAT_INC` site that distinguishes `drop` from `insert_failed`, `chaintoolong`, and `clash_resolve`. **Correction recorded:** an earlier version of this lesson showed `conntrack -S` output with `insert_failed` and `drop` incrementing together for table exhaustion. They do not — the table-full path increments `drop` only.
3. **`include/uapi/linux/netfilter.h` and `include/uapi/linux/netfilter_ipv4.h`** — **[verified against source]**. The `nf_inet_hooks` enum, the `NF_DROP`/`NF_ACCEPT`/`NF_STOLEN`/`NF_QUEUE`/`NF_REPEAT` verdicts, and the complete `nf_ip_hook_priorities` ladder reproduced in §1.
4. **`net/netfilter/nf_conntrack_proto.c`** — **[verified against source]**. The `ipv4_conntrack_ops[]` array: `ipv4_conntrack_in` at `PRE_ROUTING`/`NF_IP_PRI_CONNTRACK`, `ipv4_conntrack_local` at `LOCAL_OUT`/`NF_IP_PRI_CONNTRACK`, and `nf_confirm` at both `POST_ROUTING` and `LOCAL_IN` with `NF_IP_PRI_CONNTRACK_CONFIRM`. **Correction recorded:** the previous version of this lesson named only `POSTROUTING` as the confirmation point; `LOCAL_IN` is the other.
5. **`net/netfilter/nf_conntrack_proto_tcp.c`** — **[verified against source]**. The `tcp_timeouts[]` defaults cross-checked against the sysctl documentation, and the condition under which `IPS_ASSURED` is set (a valid ACK in `ESTABLISHED` after `SYN_RECV`), which is what makes an entry ineligible for early drop.
6. **`net/netfilter/nf_conntrack_standalone.c` and `net/netfilter/nf_conntrack_netlink.c`, with `include/uapi/linux/netfilter/nfnetlink_conntrack.h`** — **[verified against source]**. The exact `/proc/net/stat/nf_conntrack` header line, the fact that `new`, `ignore`, and `delete` are printed as literal zeros, and the ten `CTA_STATS_*` attributes that `ctnetlink_ct_stat_cpu_fill_info()` actually emits — which is what `conntrack -S` prints.
7. **`include/net/netfilter/nf_nat.h`** — **[verified against source]**. `struct nf_conn_nat`, `enum nf_nat_manip_type`, and the `HOOK2MANIP` macro that pins SRC manipulation to `POST_ROUTING`/`LOCAL_IN` and DST to the others.
8. **`conntrack-tools` — `src/conntrack.c`** — https://git.netfilter.org/conntrack-tools/ (upstream unreachable; read from a packaging mirror on GitHub) — **[verified against source]**. The command table (`-L`, `-G`, `-D`, `-I`, `-U`, `-E`, `-F`, `-C`, `-S`), the table names (`conntrack`, `expect`, `dying`, `unconfirmed`), the `cpu=N attr=value …` format of `-S`, and the `/proc/net/stat/nf_conntrack` fallback path.
9. **`libnetfilter_conntrack` — `src/conntrack/snprintf_default.c`** — **[verified against source]**, same caveat. The exact field order of `conntrack -L` output: protocol name and number, timeout, TCP state, original tuple, `[UNREPLIED]`, reply tuple, `[ASSURED]`, `mark=`, `use=`.
10. **glibc — `resolv/resolv.h`** — https://sourceware.org/git/glibc.git (read from the `bminor/glibc` mirror) — **[verified against source]**. `RES_TIMEOUT = 5` seconds and `RES_DFLRETRY = 2`, which is where the "exactly 5 seconds" DNS symptom comes from.

**Primary sources — Kubernetes**

11. **`pkg/proxy/iptables/proxier.go`** (kubernetes/kubernetes, master) — **[verified against source]**. The chain names (`KUBE-SERVICES`, `KUBE-NODEPORTS`, `KUBE-POSTROUTING`, `KUBE-MARK-MASQ`, `KUBE-FORWARD`, `KUBE-PROXY-FIREWALL`) and prefixes (`KUBE-SVC-`, `KUBE-SVL-`, `KUBE-EXT-`, `KUBE-FW-`, `KUBE-SEP-`); the jump points into `nat` `PREROUTING`/`OUTPUT`/`POSTROUTING`; the exact `KUBE-POSTROUTING` rules including `MASQUERADE --random-fully`; the per-endpoint `-m statistic --mode random --probability` construction and the `-j DNAT --to-destination` rule.
12. **`pkg/proxy/conntrack/sysctls.go` (master) and `cmd/kube-proxy/app/server_linux.go` (`release-1.34`), with `pkg/proxy/apis/config/v1alpha1/defaults.go`** — **[verified against source]**. `maxPerCore` = 32768, `min` = 131072, `tcpEstablishedTimeout` = 24 h, `tcpCloseWaitTimeout` = 1 h, the `numCPU`-from-sysfs rationale, and the fact that a 1 048 576 cap on the scaled value exists on master but **not** through v1.34. The `tcpCloseWaitTimeout` comment is also the inlined substance of kubernetes/kubernetes#32551.
13. **`pkg/proxy/ipvs/proxier.go`** — **[verified against source]**. The sysctls kube-proxy sets in IPVS mode (`net/ipv4/vs/conntrack=1`, `conn_reuse_mode`, `expire_nodest_conn=1`, `expire_quiescent_template=1`, `ip_forward=1`, and the ARP settings), the full ipset list (`KUBE-CLUSTER-IP`, `KUBE-LOOP-BACK`, `KUBE-NODE-PORT-TCP`, …) with their set types, and `defaultScheduler = "rr"`.
14. **KEP-3866, "Add an nftables-based kube-proxy backend"** (kubernetes/enhancements, SIG-Network, `@danwinship`) — https://github.com/kubernetes/enhancements/tree/master/keps/sig-network/3866-nftables-proxy — **[verified against source]**, including `kep.yaml`: feature gate `NFTablesProxyMode`, alpha v1.29, beta v1.31, **stable v1.33**, status `implemented`. Source for the O(1)-verdict-map-versus-O(n)-chain-walk claim, the control-plane argument (no incremental iptables updates), and the RHEL 9/10 deprecation driver.

**Real-world engineering — listed for depth, not relied upon**

15. **kubernetes/kubernetes#56903, "DNS intermittent delays of 5s"** and **weaveworks/weave#3287** — **[issue pages not fetched from this environment]**; the titles and the `insert_failed` diagnostic are from search metadata, and the *mechanism* described in §6 is reconstructed from the kernel source in entry 2 and glibc in entry 10, both verified. Check the issues directly for the discussion history and the full mitigation list.
16. **Cloudflare Blog — "Conntrack turns a blind eye to dropped SYNs"** — https://blog.cloudflare.com/conntrack-turns-a-blind-eye-to-dropped-syns/ — **[not reachable from this environment]**. Listed as further depth on how SYN handling under contention can drop packets in ways conntrack's own accounting does not reflect. No claim here depends on it.
17. **Mark Betz — "Exhausting conntrack table space crippled our k8s cluster"** — https://medium.com/@betz.mark/exhausting-conntrack-table-space-crippled-our-k8s-cluster-98564f6f34e0 — **[not reachable from this environment]**. The widely cited independent account of a node-global conntrack table filling in production Kubernetes; its shape matches §5–6. Read it for the narrative, not for numbers this lesson relies on.
18. **Cilium documentation — kube-proxy replacement, socket LB, and the eBPF datapath** — https://docs.cilium.io/ — **[not reachable from this environment]**. The mechanism summarised in §8 is expanded, with the actual BPF programs, in [lesson 08](08-ebpf.md), which verifies it against Cilium's own source.
