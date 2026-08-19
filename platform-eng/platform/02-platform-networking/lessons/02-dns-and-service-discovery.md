---
lesson: "A02.2"
title: "DNS and service discovery"
module: "A-02"
concept: "DNS failure amplification"
status: not-started
est_time: "5 hrs"
prev: "01-tcpip-and-the-packet-path.md"
next: "03-load-balancing.md"
artifacts: ["5s-timeout repro histogram", "ndots query-count calc", "NodeLocal DNSCache tail-latency before/after", "resolver-behavior comparison (glibc/musl/Go)"]
sources: 14
---

# A02.2 · DNS and service discovery

> **Concept.** DNS sits in the request path of everything, so its failure modes are amplifiers — an ndots search-storm or a single dropped UDP query turns into fleet-wide tail latency, and the fix is almost always caching topology, not record content.
>
> Module: [🌐 Platform networking depth](../README.md) · Track A — Platform excellence

## Where this fits

Lesson 01 established that a correctly-delivered packet is governed by a kernel pipeline with a nameable bottleneck at every stage — but before any of that machinery runs, the client has to resolve a name to an address. This lesson picks up exactly there: DNS is the hop that happens *before* the TCP handshake, on the critical path of every connection, every retry, and every service startup, and it inherits the same discipline — name the exact mechanism and the exact command, don't wave at "DNS is slow." It unlocks lesson 03 (load balancing), where you'll see the same push-vs-poll discovery tension resurface as VIPs and health checks instead of records and TTLs.

## Why this matters

DNS is the most under-respected item in the request path. It resolves before every connection, it is on the critical path of service startup and of every retry, and its failures are *amplified*: a resolver that flaps for 30 seconds browns out every service that calls it during that window; a negative-cache TTL set wrong turns a 30-second blip into a 30-minute outage; a conntrack race that drops one UDP packet in a hundred injects a hard 5-second stall into a random 1% of all lookups. At the synchronized start of a 512-GPU training job, that same 5-second stall on the slowest rank's peer lookup delays the entire collective. Staff-level fluency here is knowing *why* these amplify and naming the caching-topology fix, not reciting record types.

## What's new here (calibration)

- You already know A/AAAA/CNAME/SRV/PTR record types, recursion vs iteration, TTL and CNAME chains, `dig` basics, and that Kubernetes ships CoreDNS serving records for Services and pods — none of that is re-taught here.
- New: the *exact* resolver algorithm — glibc's, musl's, and Go's — read from their source, because the same symptom has three different signatures and three different fixes depending on which one is in the container.
- New: the ndots expansion as arithmetic you can do on a whiteboard, extended into a CoreDNS capacity-planning formula.
- New: DNS scaling and failure as a shared-dependency, multi-tenant blast-radius problem — CoreDNS is infrastructure on the same tier as etcd, not a library call.
- New: the precise asymmetry between how SERVFAIL, NXDOMAIN, and a dropped non-response are each cached and retried — including the correction that **CoreDNS does cache SERVFAIL, for 5 seconds by default**.
- Assumed from `modules/01b-linux-internals/lessons/07-networking-datapath-conntrack.md`: what a conntrack entry is, that NAT rides on conntrack, and that confirmation happens last in the hook chain. This lesson uses those facts; it does not re-derive them.

## Core concepts

### 1. Why DNS is an amplifier, structurally

Three properties make DNS behave unlike any other dependency in your stack.

**It is *upstream* of everything.** Nothing retries around it, because there is nothing to retry *to* — you cannot open a connection to a service whose address you don't have. A 200 ms regression in your database adds 200 ms to the requests that touch the database. A 200 ms regression in DNS adds 200 ms to every first connection in the fleet, including the ones to the database, the object store, the metrics endpoint, and the identity provider.

**Its retries are timers, not signals.** TCP tells you about failure with an RST or an ICMP unreachable. UDP DNS tells you nothing: a dropped query is indistinguishable from a slow server, so the resolver's only recourse is to wait out a **fixed timeout**. That timeout is measured in *seconds* while everything else in your request budget is measured in milliseconds. This is why DNS failures produce bimodal latency histograms with a spike at a round number, rather than a smear.

**Its caching is a distributed system you did not design.** The answer any given client uses is a function of that client's stub-resolver cache, its libc's behaviour, the node-local cache if you have one, CoreDNS's cache, the upstream forwarder's cache, and the authoritative TTL — each with its own expiry. When you change a record, you are not making a change; you are starting a wave that propagates over the union of those TTLs.

Add these up and the failure mode is always the same shape: **a small, local, transient fault becomes a large, global, sustained one.**

### 2. The Kubernetes resolv.conf, exactly

Every pod in `ClusterFirst` DNS policy gets a `/etc/resolv.conf` written by the kubelet. Here is a real one from a pod in namespace `training`, on a cluster with domain `cluster.local`, running in a VPC that injects its own search domain:

```
nameserver 10.96.0.10
search training.svc.cluster.local svc.cluster.local cluster.local eu-west-1.compute.internal
options ndots:5
```

Three lines, each of which is load-bearing:

**`nameserver 10.96.0.10`** is the `kube-dns` Service ClusterIP — a *virtual* address with no listener behind it. Every query to it is DNATed by the node's dataplane to one of the CoreDNS pod IPs. That DNAT is where §5's race lives. Note also that both glibc and Go cap the nameserver list at **3** (`MaxDNSNameservers = 3` in `pkg/apis/core/validation/validation.go`; glibc's `MAXNS` is likewise 3) — extra entries are silently ignored.

**`search ...`** is built by the kubelet as `<namespace>.svc.<clusterDomain>`, `svc.<clusterDomain>`, `<clusterDomain>` (`pkg/kubelet/network/dns/dns.go`), followed by whatever the node's own `/etc/resolv.conf` carried. Kubernetes validates the result against `MaxDNSSearchPaths = 32` and `MaxDNSSearchListChars = 2048`.

**`options ndots:5`** is the kubelet default, hardcoded as `defaultDNSOptions = []string{"ndots:5"}` in the same file. Note how far it is from every resolver's own default, which is **`ndots:1`** (glibc `res_init.c`: `parser->template.ndots = 1`; musl `resolvconf.c`: `conf->ndots = 1`; Go `dnsconfig_unix.go`: `ndots: 1`). Kubernetes raises it to 5 so that a pod can say `payments` or `payments.billing` or `payments.billing.svc` and have all three work. **That convenience is the tax.**

### 3. ndots, and why one lookup becomes eight

The algorithm is identical in glibc, musl, and Go, and it is small enough to state exactly. Given a name:

1. Count the dots in the name.
2. **If the name ends in a dot** (it is already absolute), query it and *only* it. Never search.
3. **Else if `dots >= ndots`**, query the name as-is first; if that fails, walk the search list.
4. **Else** (`dots < ndots`), walk the search list first, one suffix at a time, and only if all of them fail, query the name as-is.

*(glibc: `resolv/res_query.c`, `__res_context_search` — `if (dots >= statp->ndots || trailing_dot)`. musl: `src/network/lookup_name.c` — `if (dots >= conf.ndots || name[l-1]=='.') *search = 0;`. Go: `src/net/dnsclient_unix.go`, `nameList` — `hasNdots := CountString(name, '.') >= conf.ndots`.)*

Each "query" in that list is really **two** wire queries — A and AAAA — because `getaddrinfo(AF_UNSPEC)` needs both address families. So the query count for a relative name is:

```
   wire queries = (N_search + 1) × 2        where N_search = len(search list)
```

Here is what that does to a perfectly ordinary external hostname:

```
   RESOLVING  s3.eu-west-1.amazonaws.com  FROM A POD
   ndots:5, search list of 4.  The name has 3 dots → 3 < 5 → it is RELATIVE.
   ═══════════════════════════════════════════════════════════════════════════

   pod                     CoreDNS (via ClusterIP DNAT)          upstream
   ───                     ─────────────────────────────          ────────
   1  A     s3.eu-west-1.amazonaws.com.training.svc.cluster.local ──▶ kubernetes
   2  AAAA  ────────────────────"───────────────────────────────  ──▶  plugin
                    ◀── NXDOMAIN ×2                                    answers
                                                                       NXDOMAIN
   3  A     s3.eu-west-1.amazonaws.com.svc.cluster.local          ──▶  locally,
   4  AAAA  ────────────────────"───────────────────────────────       no upstream
                    ◀── NXDOMAIN ×2                                    hop needed

   5  A     s3.eu-west-1.amazonaws.com.cluster.local              ──▶
   6  AAAA  ────────────────────"───────────────────────────────
                    ◀── NXDOMAIN ×2

   7  A     s3.eu-west-1.amazonaws.com.eu-west-1.compute.internal ──▶ forward
   8  AAAA  ────────────────────"───────────────────────────────  ──▶ plugin →
                    ◀── NXDOMAIN ×2                                   VPC resolver
                                                                      (REAL NETWORK
                                                                       ROUND TRIP)
   9  A     s3.eu-west-1.amazonaws.com.                           ──▶ forward →
  10  AAAA  ────────────────────"───────────────────────────────  ──▶ VPC resolver
                    ◀── 52.218.x.x  ✓                                 ✓

   ═══════════════════════════════════════════════════════════════════════════
   TOTAL: 10 wire queries.  8 of them NXDOMAIN.  ONE was the answer (plus its
   AAAA sibling).  Queries 7–10 each cross the node boundary AND the VPC
   boundary; 1–6 are answered inside CoreDNS but still cross the node boundary.

   SAME NAME WITH A TRAILING DOT — s3.eu-west-1.amazonaws.com.
   ───────────────────────────────────────────────────────────
   1  A     s3.eu-west-1.amazonaws.com.  ──▶ ✓
   2  AAAA  ──────────"─────────────────  ──▶ ✓
   TOTAL: 2.  A FIVEFOLD REDUCTION FROM ONE CHARACTER.
```

**The in-cluster case is not exempt, it is just luckier.** `payments.billing.svc.cluster.local` has 4 dots — still fewer than 5 — so it is *also* relative and the resolver tries `payments.billing.svc.cluster.local.training.svc.cluster.local` first. That fails at CoreDNS's `kubernetes` plugin without leaving the node, so it is cheap in latency terms, but it is not free in QPS terms.

**The capacity-planning formula.** CoreDNS's load is not "distinct names your fleet resolves." It is:

```
   QPS_coredns  =  R_conn × (N_search + 1) × 2 × (1 − h_local)

     R_conn    = new outbound connections per second across the fleet
                 (this is the number people forget: connection churn,
                  not request rate — HTTP keep-alive collapses it)
     N_search  = search list length (4 in the resolv.conf above)
     h_local   = node-local cache hit ratio (0 if you have no NodeLocal
                 DNSCache; typically 0.8–0.95 if you do)
```

**Worked, for a mid-size cluster.** 400 nodes × 40 pods = 16,000 pods. Assume each pod opens 2 new outbound connections per second on average (a mix of gRPC channel re-establishment, sidecar-less HTTP without pooling, and health checks):

```
   R_conn   = 16,000 × 2                = 32,000 conn/s
   naive DNS QPS (no ndots, no A/AAAA)  = 32,000 q/s
   with ndots amplification (N=4)       = 32,000 × 5 × 2 = 320,000 q/s
   with NodeLocal DNSCache at h = 0.9   = 320,000 × 0.1  =  32,000 q/s

   A CoreDNS pod comfortably serves on the order of 10k q/s for cached
   answers on a modern core (measure yours — it depends on the plugin
   chain and whether answers come from the kubernetes plugin's local
   store or from `forward`).

   → without the local cache:  320,000 / 10,000 = 32 CoreDNS replicas
   → with the local cache:      32,000 / 10,000 =  3 CoreDNS replicas
```

**That order-of-magnitude gap is the entire argument for NodeLocal DNSCache, and it is a capacity number, not an aesthetic preference.** Note also what the formula says about application behaviour: halving connection churn (keep-alive, connection pools, gRPC channel reuse) halves DNS load exactly as effectively as doubling the cache hit rate, and is usually cheaper.

### 4. The three resolvers, side by side

**This is the section that decides whether your fix works.** The same pod-level symptom has three different mechanisms depending on the libc or runtime in the image, and the classic glibc remedy is a no-op on two of the three.

| Property | **glibc** (Debian/Ubuntu/RHEL images) | **musl** (Alpine images) | **Go** (`net` pure-Go resolver) |
|---|---|---|---|
| Source of truth | `resolv/res_init.c`, `res_send.c`, `res_query.c` | `src/network/resolvconf.c`, `res_msend.c`, `lookup_name.c` | `src/net/dnsconfig_unix.go`, `dnsclient_unix.go` |
| Default `ndots` | 1 | 1 | 1 |
| Default timeout | `RES_TIMEOUT` = **5 s** | `timeout` = **5 s** *total* | 5 s per query |
| Default attempts | `RES_DFLRETRY` = **2** | **2** | 2 |
| **Retransmit interval** | **5 s** (`seconds = retrans << ns`) | **timeout / attempts = 2.5 s** | 5 s |
| A + AAAA sent | in **parallel**, on **one** UDP socket | in **parallel**, on **one** UDP socket | in **parallel**, on **separate** sockets (one goroutine + one `Dial` each) |
| `options single-request` | supported (`RES_SNGLKUP`) | **not supported** | supported (`conf.singleRequest`) |
| `options single-request-reopen` | supported (`RES_SNGLKUPREOP`) | **not supported** | not supported |
| `options use-vc` (force TCP) | supported | not supported | supported (`conf.useTCP`) |
| Adaptive fallback | **yes** — on a timeout after receiving one of two answers it self-switches to single-request, then to single-request-reopen (`res_send.c`) | no | no |
| Search-list order for `dots >= ndots` | as-is first, then search | **search first, always**, then as-is | as-is first, then search |

Read three consequences off that table:

1. **The stall duration tells you the libc.** A histogram spike at **exactly 5.0 s** is glibc or Go. A spike at **2.5 s** (and a second one near 5.0 s when the retry also fails) is musl. If someone reports "the classic 5-second DNS bug" on an Alpine image, the timing does not match the story and you should re-measure before acting.
2. **`single-request-reopen` does not exist on Alpine.** Adding `options single-request-reopen` to a musl container's `dnsConfig` is silently ignored — musl's option parser only knows `ndots`, `attempts`, and `timeout`. The remedies that *do* work there are structural: NodeLocal DNSCache, an eBPF dataplane, or forcing TCP at a layer musl doesn't control.
3. **Go is structurally immune to the same-socket variant of the race** because each query gets its own socket and therefore its own conntrack tuple — but it is fully exposed to the ndots tax and to conntrack *table pressure*, and a Go binary compiled with cgo may use glibc's resolver instead (`GODEBUG=netdns=go` forces the pure-Go path, `netdns=cgo` forces the libc path; `GODEBUG=netdns=2` makes Go log which it chose).

### 5. The 5-second stall: the exact mechanism

This is the canonical staff "intermittent latency" story, and it is worth getting exactly right because the wrong version of it leads to the wrong fix.

**What the resolver does.** glibc's `getaddrinfo` for `AF_UNSPEC` sends the A query and the AAAA query back to back from **the same UDP socket** — same source IP, same source port, same destination — then `poll()`s for both answers. This is deliberate (it halves latency in the common case) and is documented in `res_send.c`'s own comments as the default, with `RES_SNGLKUP`/`RES_SNGLKUPREOP` as escape hatches for "broken name servers … [that] don't handle two outstanding requests from the same source."

**What the kernel does to them.** Both packets are `NEW` to conntrack. Both traverse the `nat` table and get DNATed from the kube-dns ClusterIP to a CoreDNS pod IP. Per 01b.7's central fact, **the conntrack entry is not inserted into the hash table until confirmation, which runs dead last in the hook chain.** So two packets from the same socket can both be in flight through the hooks at once, both allocate an unconfirmed entry, and both attempt to insert. Whichever confirms second finds a conflicting entry already present, `insert_failed` is incremented, and **the packet is dropped silently** — no ICMP, no error to the sender.

**What the application sees.** One of the two answers never arrives. glibc waits out its retransmit timer — `RES_TIMEOUT = 5` seconds, computed in `res_send.c` as `seconds = statp->retrans << ns` — and then retries. The retry almost always succeeds, because the race needs simultaneity. So the user-visible event is: **the name resolved correctly, and it took 5.000 seconds.**

```
   TIMELINE OF ONE STALLED LOOKUP  (glibc, ndots-expanded name, busy node)
   ═══════════════════════════════════════════════════════════════════════

   t=0.000  app: getaddrinfo("payments.billing", AF_UNSPEC)
   t=0.000  glibc opens ONE UDP socket, srcport 41337
            ├─ sendto()  A     payments.billing.training.svc.cluster.local
            └─ sendto()  AAAA  payments.billing.training.svc.cluster.local
                          (microseconds apart, same 5-tuple except QTYPE,
                           which conntrack cannot see — it is DNS payload)

   t=0.000+ kernel, packet 1:  conntrack MISS → alloc unconfirmed entry
                               nat: DNAT 10.96.0.10:53 → 10.244.3.9:53
                               ... hooks ... nf_confirm → INSERT ✓
   t=0.000+ kernel, packet 2:  conntrack MISS → alloc unconfirmed entry
                               nat: DNAT 10.96.0.10:53 → 10.244.3.9:53
                               ... hooks ... nf_confirm → CLASH ✗
                                   insert_failed++   packet FREED
                                   ◀── NOTHING is sent back. Not an ICMP,
                                       not an RST. The query evaporates.

   t=0.002  CoreDNS answers the A query.  glibc has one of two answers.
            poll() keeps waiting for the AAAA.

   t≈0.002 ─────────────── 4.998 seconds of nothing ───────────────▶

   t=5.000  glibc's RES_TIMEOUT expires.
            Because it already received ONE answer, glibc concludes the
            server may be "broken" and sets RES_F_SNGLKUP for this
            resolver state, then retries SERIALLY.
   t=5.002  AAAA answered.  getaddrinfo returns.  Total: 5.002 s.

   THE HISTOGRAM SIGNATURE: a dense mode at ~2 ms and a razor-thin spike
   at 5.00 s.  A smear between 1 s and 5 s is a DIFFERENT problem (an
   overloaded CoreDNS, or upstream latency) — do not treat them the same.
```

**Confirming it, with real commands and what to read:**

```bash
# 1. Is conntrack actually failing inserts on this node?
$ conntrack -S | head -4
cpu=0   found=0 invalid=1832 insert=0 insert_failed=417 drop=417 \
        early_drop=0 error=0 search_restart=9012 clash_resolve=1201
cpu=1   found=0 invalid=1799 insert=0 insert_failed=402 drop=402 \
        early_drop=0 error=0 search_restart=8877 clash_resolve=1184
#                                    ^^^^^^^^^^^^^^^^ ^^^^^^^^
#  insert_failed climbing in lockstep with drop is THE fingerprint.
#  Note insert_failed and drop being equal: every failed insert cost a packet.
#  (Representative values; run it twice 60 s apart and diff — absolute
#   numbers since boot mean nothing.)

# 2. Are the 5 s events correlated in time?
$ for i in $(seq 200); do
>   /usr/bin/time -f '%e' getent hosts payments.billing 2>&1 >/dev/null
> done | sort -n | uniq -c | tail -5
    193 0.00
      1 0.01
      6 5.00
#         ^^^^ 6/200 = 3 % of lookups took exactly 5.00 s.

# 3. Which resolver is actually in play?
$ ldd /proc/1/root/usr/bin/myapp 2>/dev/null | grep -E 'libc|musl'
        libc.musl-x86_64.so.1 => /lib/ld-musl-x86_64.so.1
#  → Alpine. The 5.00 s figure above would then be SUSPICIOUS: musl
#    retransmits at timeout/attempts = 2.5 s. Re-measure before acting.
```

### 6. The fix ladder, ordered by how structural it is

**Rung 0 — remove the queries.** The cheapest query is the one not sent. Fully-qualify external names with a trailing dot (drops 10 queries to 2), and set a per-workload `ndots` where you control the code:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: dataloader
spec:
  dnsPolicy: ClusterFirst
  dnsConfig:
    options:
      - name: ndots
        value: "2"          # in-cluster short names of the form
                            # <svc>.<ns> still work; anything with 2+
                            # dots is tried absolute-first
      - name: single-request-reopen   # glibc ONLY — no-op on Alpine
    searches: []            # optional: trim inherited VPC search domains
  containers:
    - name: app
      image: ghcr.io/example/dataloader:1.4
```

The trade is stated plainly: with `ndots:2`, a pod that calls the bare short name `payments` (0 dots) still searches, and `payments.billing` (1 dot) still searches, but `payments.billing.svc` (2 dots) is now tried absolute-first and will **fail**, because the absolute name `payments.billing.svc.` does not exist. Lower `ndots` only where you know the calling code's naming convention.

**Rung 1 — CoreDNS `autopath`.** Move the search-list walk server-side. `autopath` watches pods to learn the client's namespace, then, when it sees a query matching the first search-path element, it follows the chain itself and returns the first non-NXDOMAIN answer, with a CNAME from the asked-for name to the real one. Ten wire queries become two.

```
. {
    kubernetes cluster.local in-addr.arpa ip6.arpa {
        pods verified          # autopath REQUIRES pods verified mode
        fallthrough in-addr.arpa ip6.arpa
        ttl 30
    }
    autopath @kubernetes       # take the search path from the kubernetes plugin
    cache 30
    forward . /etc/resolv.conf { max_concurrent 1000 }
    loop
    reload
    loadbalance
}
```

Read the caveats from the plugin's own "Bugs" section before you ship it: it resolves the client IP to a pod via the API watch, so a pod IP recycled into a *different namespace* faster than the watch propagates gets the wrong search path; it does not work for Windows-node pods; and **if the final answer is negative anyway, the client still walks every path manually**, so autopath saves nothing on a genuinely nonexistent name. `pods verified` also costs CoreDNS a full pod watch, which is real memory on a large cluster.

**Rung 2 — NodeLocal DNSCache.** The structural fix. A DaemonSet puts a caching CoreDNS instance on every node, listening on a link-local address (the convention is `169.254.20.10`; the docs require only that it not collide with anything in the cluster). Pods talk to it over loopback-ish local delivery — **no ClusterIP, therefore no DNAT, therefore no conntrack entry, therefore no race** for cache hits. Cache misses for cluster names are forwarded to the kube-dns Service **over TCP**.

The `force_tcp` is not decoration; it is the mechanism, and it is right there in the shipped manifest (`cluster/addons/dns/nodelocaldns/nodelocaldns.yaml`):

```
__PILLAR__DNS__DOMAIN__:53 {
    errors
    cache {
            success 9984 30       # 9984 entries, 30 s max TTL
            denial  9984 5        # negative answers capped at 5 s
    }
    reload
    loop
    bind __PILLAR__LOCAL__DNS__ __PILLAR__DNS__SERVER__
    forward . __PILLAR__CLUSTER__DNS__ {
            force_tcp             # ◀── THE FIX. TCP conntrack entries are
    }                             #     torn down on close; UDP entries linger
    prometheus :9253              #     for nf_conntrack_udp_timeout (30 s
    health __PILLAR__LOCAL__DNS__:8080   #  default) and there is no
}                                 #     handshake to race.
```

Two things worth noticing in that config that people miss:

- **The `.:53` block — the one that handles *external* names — does not set `force_tcp`.** It forwards to the upstream resolvers over UDP. So NodeLocal DNSCache removes the race for cluster-name misses but not necessarily for external-name misses. It still helps enormously, because those queries now originate from the node's own network namespace rather than through a Service DNAT, but "NodeLocal DNSCache eliminates the race" is too strong.
- **Denial caching at 5 s.** Every one of those ndots NXDOMAINs is cached locally for 5 seconds, which is where most of the 90 % hit rate in §3's arithmetic actually comes from.

**Rung 3 — an eBPF dataplane.** Cilium's kube-proxy replacement performs service translation at the socket layer (`connect(2)`/`sendmsg(2)` are rewritten before a packet exists) rather than via `nat`-table DNAT on a formed packet. There is no DNAT, so there is no NAT-conntrack insert to race. This removes the mechanism rather than avoiding it — see lesson 05 for how socket-level translation works and what it costs.

### 7. Negative caching: the taxonomy that decides your incident's shape

The single most useful distinction in DNS operations is **what each kind of failure does to load**. Three cases, three completely different incident shapes:

| Response | Cached? | Effect on upstream load during a fault | Incident shape |
|---|---|---|---|
| **NXDOMAIN** (name does not exist) | yes — against the zone's SOA minimum, capped by the cache plugin's denial TTL (CoreDNS: max **1800 s**, min **5 s**, default cap 3600/1800 per the `cache` README) | **falls** — clients stop asking | a *stale* outage: the record now exists but nobody can see it until the negative TTL expires |
| **SERVFAIL** (resolution failed) | **yes in CoreDNS — for 5 s by default**, tunable via `servfail DURATION`, hard-capped at 5 minutes | **rises**, but with a 5-second damper | a retry storm, damped |
| **No response at all** (dropped UDP) | nothing to cache | **rises**, undamped, and each client burns a full timeout first | the 5-second-stall incident of §5 |

**Correction worth flagging:** the widely repeated claim that "SERVFAIL is never cached" is not true of CoreDNS. The `cache` plugin's README states it plainly: "`servfail` cache SERVFAIL responses for **DURATION**. Setting **DURATION** to 0 will disable caching of SERVFAIL responses. If this option is not set, SERVFAIL responses will be cached for 5 seconds. **DURATION** may not be greater than 5 minutes." That 5-second damper is deliberately short precisely so a transient upstream fault is not pinned as authoritative — but it does mean CoreDNS gives you a knob to widen the damper during a known-bad upstream, and that the load-amplification story is "damped, not absent."

The classic negative-caching trap runs the other way. Set an aggressive denial TTL to protect CoreDNS from an NXDOMAIN storm, then deploy a new Service, and every pod that happened to look it up during the pre-deploy window is stuck with a cached NXDOMAIN for the full TTL. **Negative TTL is your rollout latency floor.** Keep it small (the shipped Kubernetes NodeLocal config uses 5 s for denials against 30 s for successes, which is exactly this reasoning), and prefer more caching *layers* over longer caching *durations*.

### 8. CoreDNS as a shared dependency

CoreDNS is a **plugin chain**, and the order in the Corefile is the order of execution. Here is the actual default Kubernetes ships (`cluster/addons/dns/coredns/coredns.yaml.base`), annotated:

```
.:53 {
    errors                       # log errors to stdout
    health {
        lameduck 5s              # on SIGTERM, keep answering /health OK for 5 s
    }                            # so the endpoint controller can drain us first
    ready                        # :8181/ready — used as the readinessProbe
    kubernetes cluster.local in-addr.arpa ip6.arpa {
        pods insecure            # answer pod A records without verifying the pod
                                 # exists. 'verified' costs a pod watch; 'disabled'
                                 # turns pod records off. autopath needs 'verified'.
        fallthrough in-addr.arpa ip6.arpa   # if we don't own this reverse name,
                                 # pass it down the chain instead of NXDOMAIN
        ttl 30                   # TTL on records we generate. Default is 5 s;
    }                            # k8s ships 30 s. Max allowed is 3600.
    prometheus :9153             # metrics endpoint
    forward . /etc/resolv.conf {
        max_concurrent 1000      # REFUSED beyond 1000 in-flight upstream queries.
    }                            # Each concurrent query costs ~2 KB of memory.
    cache 30                     # max TTL 30 s for both success and denial here
    loop                         # startup loop detection: sends itself a random
                                 # HINFO probe; halts the process if it sees it
                                 # more than twice. This is what catches the
                                 # "forward . /etc/resolv.conf points at me" bug.
    reload                       # watch the Corefile and reload on change
    loadbalance                  # shuffle A/AAAA RRsets in responses
}
```

**What each default actually buys you, with the numbers:**

- **`cache`** without arguments would cap at 3600 s for NOERROR and 1800 s for denials, with a 5 s minimum and a store of 256 shards × 39 items = **9984 entries per cache type** (success and denial are separate caches, so 19,968 entries total by default). Kubernetes overrides the cap to 30 s. Eviction is per-shard, so evictions begin before the nominal capacity is reached — if `coredns_cache_evictions_total` is climbing while `coredns_cache_entries` is well below 9984, that's shard skew, and the fix is a larger explicit `success`/`denial` capacity, rounded down to a multiple of 256.
- **`forward`'s health checking** is in-band: when an exchange errors, it starts probing that upstream every **0.5 s** with a recursive `. IN NS` query until it answers. `max_fails` defaults to **2** consecutive failures before an upstream is marked down. Read timeout is **2 s** per upstream. Upstream selection policy defaults to **`random`**. When *all* upstreams are down it sprays to a random one anyway (and increments `coredns_forward_healthcheck_broken_total`) — the reasoning being that health checking itself has evidently failed. `failfast_all_unhealthy_upstreams` changes that to an immediate SERVFAIL.
- **`health { lameduck 5s }`** is the graceful-shutdown contract. On `SIGTERM`, CoreDNS keeps reporting healthy for 5 s so that endpoint removal propagates before it stops answering. Without it, a rolling CoreDNS update drops queries for however long endpoint propagation takes — which on a large cluster is the very thing you were trying to avoid.

**The operational framing.** Treat CoreDNS the way you treat etcd or a shared cache tier, because it has the same blast radius:

- **Never one replica.** A single-replica CoreDNS makes every pod in the cluster share one process's fate and one node's kernel.
- **Scale on QPS, not CPU, and don't rely on reactive autoscaling alone.** The HPA reacts *after* the load arrives; a rollout that recreates 3,000 pods produces its DNS spike in the first 20 seconds. The usual answer is `cluster-proportional-autoscaler` (replicas as a function of node/core count) as the floor, with an HPA above it.
- **Watch the four metrics that matter**: `coredns_dns_requests_total` (rate, by `type` — a rising `AAAA` share with no IPv6 in the cluster is the ndots tax made visible), `coredns_dns_responses_total{rcode="NXDOMAIN"}` (the same tax from the other side), `coredns_dns_request_duration_seconds` p99, and `coredns_forward_max_concurrent_rejects_total` (you are REFUSING queries; raise `max_concurrent` or add replicas).

### 9. The records Kubernetes actually generates

Worth having exactly, because the difference between a ClusterIP Service and a headless one changes what the client does next.

For a Service `payments` in namespace `billing`, cluster domain `cluster.local` (from the Kubernetes DNS-based service discovery specification in `kubernetes/dns`):

| Kind | Name | Answer |
|---|---|---|
| ClusterIP A | `payments.billing.svc.cluster.local.` | the **ClusterIP** — one address, virtual, no listener |
| SRV (per named port) | `_grpc._tcp.payments.billing.svc.cluster.local.` | `<prio> <weight> <port> payments.billing.svc.cluster.local.` |
| PTR | `<d>.<c>.<b>.<a>.in-addr.arpa.` | `payments.billing.svc.cluster.local.` |
| **Headless** A | `payments.billing.svc.cluster.local.` | **every ready endpoint's pod IP**, as an RRset |
| Per-pod A (headless) | `<hostname>.payments.billing.svc.cluster.local.` | that one endpoint's IP |
| ExternalName | `payments.billing.svc.cluster.local.` | a `CNAME` to the external name |

**The headless-Service consequence people get wrong.** A headless Service's answer is an RRset of pod IPs, so the *client* is now doing the load balancing and the *client's* cache is now the endpoint list. That is exactly the poll-based discovery model in §10, with all of its staleness properties, and it is why a StatefulSet scale-down produces connection errors at clients that cached the RRset. "Headless" changes what the answer contains; it does not change how the answer is resolved, cached, or invalidated, and it does not make the lookup cheaper — the query still walks the same ndots expansion and the same CoreDNS path.

### 10. Push vs poll: why DNS is the wrong primitive for fast failover

Every service-discovery system sits somewhere on one axis: **does the client learn about a change because someone told it, or because it asked again?**

```
   THE DISCOVERY AXIS
   ══════════════════════════════════════════════════════════════════════

   POLL / TTL-CACHED                              PUSH / WATCH
   (DNS, most cloud metadata)                     (etcd watch, Consul,
                                                   xDS, K8s EndpointSlice
                                                   informers)
   ┌────────────────────────────┐                ┌────────────────────────┐
   │ client asks, caches for T  │                │ client subscribes once │
   │                            │                │                        │
   │ staleness window: [0, T]   │                │ staleness window:      │
   │   — uniformly distributed  │                │   [0, propagation]     │
   │   — you cannot shorten it  │                │   — typically ms       │
   │     without a query storm  │                │                        │
   │                            │                │ cost: a long-lived     │
   │ cost: O(clients × 1/T)     │                │ connection per client, │
   │ queries per second         │                │ and the server holds   │
   │                            │                │ per-client state       │
   │ failure mode: clients keep │                │ failure mode: the      │
   │ dialing dead endpoints for │                │ control plane is now a │
   │ up to T                    │                │ hard dependency, and a │
   │                            │                │ reconnect storm after  │
   │                            │                │ a control-plane blip   │
   │                            │                │ is a thundering herd   │
   └────────────────────────────┘                └────────────────────────┘
              │                                              │
              └────────────── THE TRAP ──────────────────────┘
     "Just set TTL to 1 s."  Now every client re-queries every second:
     16,000 pods × 1 q/s × (N+1) × 2 = 160,000 q/s of pure overhead,
     and the staleness window is STILL up to 1 s plus however long the
     client library caches on top of the DNS TTL — which Java's
     `networkaddress.cache.ttl` historically defaulted to FOREVER for
     successful lookups under a security manager. TTL is a hint you do
     not control.
```

**The rule that falls out:** use DNS for *bootstrapping* and for *coarse* discovery; use a VIP/L4 load balancer or a watch-based control plane for anything that must fail over in less than a TTL. That is precisely why Kubernetes gives a Service a stable ClusterIP rather than telling clients to re-resolve: the VIP is a *level of indirection whose backing set changes synchronously in the dataplane*, so an endpoint can be removed without any client re-resolving anything. Lesson 03 is the same argument one layer up, and lesson 05 is how the VIP indirection is actually implemented.

### 11. The GPU-fleet angle

Multi-node training resolves peer names at job start — the rendezvous endpoint, rank-0's address, and (with a StatefulSet plus a headless Service) each rank's stable per-pod name. Three properties combine badly:

1. **All ranks resolve at the same instant.** A 512-GPU job is 64 pods starting within a second of each other, each doing its ndots expansion. That is a synchronized burst, not a steady rate, and it lands on CoreDNS as a spike an HPA cannot react to in time.
2. **A collective is a barrier.** The job starts when the *slowest* rank finishes rendezvous. One rank hitting the 5-second stall delays all 512 GPUs by 5 seconds. At a representative $30–$40/hr for an 8-GPU node (verify your own rate — these move), 64 nodes idle for 5 seconds is small in isolation and enormous when it happens on every restart of a job that crash-loops.
3. **Restarts multiply it.** A job that OOMs and restarts every 20 minutes re-pays the rendezvous DNS burst every 20 minutes, and a flaky rank that restarts alone re-resolves against a cold local cache.

The standard hardening is unglamorous and effective: NodeLocal DNSCache on every GPU node (so the burst is served from local memory), fully-qualified names in the launcher's environment so no expansion happens at all, and a rendezvous backend that does not depend on DNS at steady state. The fabric-side mechanics of the collective itself are module 09's territory; what you own here is that the *name resolution* preceding it is not on the critical path of a barrier.

## Perspectives

**Client-library.** glibc, musl, and Go are three different resolvers with three different failure signatures under identical network conditions, and the diagnostic value is in the *timing*: a spike at 5.0 s says glibc or Go, a spike at 2.5 s says musl. A staff engineer identifies the runtime on the affected pod (`ldd`, or `GODEBUG=netdns=2` for a Go binary) before proposing a fix, because `single-request-reopen` — the reflexive answer — is parsed by glibc and Go, ignored entirely by musl, and irrelevant to Go's already-separate sockets.

**Multi-tenant blast-radius.** CoreDNS is not "a pod that answers DNS," it is a shared dependency on the same tier as etcd. Its capacity, its rollout strategy (`lameduck`), and its replica floor should be reasoned about the way you reason about any fleet-wide SPOF. The specific trap is autoscaler lag: DNS load is driven by *connection churn*, which spikes precisely during the rollouts and scale events when an HPA is least able to respond.

**Cache-topology.** The right mental model is layers, not durations. Every additional cache layer (stub → node-local → CoreDNS → forwarder) multiplies your effective hit rate while leaving the *staleness ceiling* at the authoritative TTL. Every additional second of TTL raises the hit rate too, but it also raises the staleness ceiling and therefore your rollout latency floor. Prefer more layers over longer TTLs — the shipped NodeLocal config's 30 s success / 5 s denial split is exactly that principle in configuration form.

**Economics / scale.** The ndots tax is a multiplier, not a fixed cost: `(N_search + 1) × 2` on top of connection churn. At the mid-size cluster in §3 that is the difference between 3 and 32 CoreDNS replicas, and the cheapest lever is not more replicas — it is HTTP keep-alive and connection pooling in the applications, which reduces `R_conn` at the source and reduces conntrack pressure at the same time.

## Real-world use cases

- **Kubernetes issue #56903, "DNS intermittent delays of 5s."** The upstream issue the NodeLocal DNSCache documentation itself cites as the motivation for the feature. Substance: the conntrack DNAT race on parallel A/AAAA queries from one socket, producing a hard 5-second stall equal to glibc's `RES_TIMEOUT`. This issue is what turned a libc default into a Kubernetes platform problem. *(Referenced by URL from the Kubernetes NodeLocal DNSCache task page, which was read directly; the issue thread itself was not fetched in this pass.)*
- **NodeLocal DNSCache's own motivation section** (`kubernetes/website`, `docs/tasks/administer-cluster/nodelocaldns.md`), read directly. Substance used here: that skipping iptables DNAT and connection tracking "will help reduce conntrack races and avoid UDP DNS entries filling up conntrack table"; that TCP conntrack entries are removed on connection close whereas UDP entries must time out (`nf_conntrack_udp_timeout` default 30 s); and that the local listen address should come from the link-local range `169.254.0.0/16` (or IPv6 ULA `fd00::/8`).
- **CoreDNS `autopath`'s documented bugs**, read directly from the plugin README. Substance: the pod-IP-recycled-into-another-namespace race that gives a client the wrong search path, the Windows-node incompatibility, and the fact that a genuinely negative final answer defeats the optimisation entirely because the client falls back to walking every path itself. This is a good example of a fix whose correctness envelope is narrower than its marketing.
- **The `forward` plugin's all-upstreams-down behaviour.** Substance: when every upstream is marked unhealthy, CoreDNS assumes health checking itself has failed and sprays to a random upstream anyway, incrementing `coredns_forward_healthcheck_broken_total`. This produces the counter-intuitive incident where a totally dead upstream still receives full query load — and it is why that metric belongs on the DNS dashboard.

## Worked example

**Scenario.** A 400-node cluster reports "intermittent 5-second API latency" from a Python service in namespace `training`, running on a Debian-based image. p50 is 4 ms, p99 is 5.01 s, and it is worse on the busiest nodes. Bisect it, fix it, and prove the fix with numbers.

**Step 1 — establish the shape, not the average.** An average is useless here; you need the distribution.

```bash
$ kubectl exec -n training deploy/api -- sh -c '
    for i in $(seq 500); do
      s=$(date +%s%N)
      getent hosts payments.billing >/dev/null
      e=$(date +%s%N)
      echo $(( (e-s)/1000000 ))
    done' | sort -n | awk '
      $1<10   {a++} $1<100 {b++} $1<1000 {c++} $1<4900 {d++} $1>=4900 {e++}
      END {printf "<10ms:%d  <100ms:%d  <1s:%d  <4.9s:%d  >=4.9s:%d\n",a,b-a,c-b,d-c,e}'
<10ms:487  <100ms:2  <1s:0  <4.9s:0  >=4.9s:11
```

**11 of 500 (2.2 %) at ≥ 4.9 s and nothing in between.** A bimodal distribution with an empty middle is the signature of a *fixed timer*, not of a loaded server. A loaded CoreDNS would smear.

**Step 2 — identify the resolver, because the number has to match.**

```bash
$ kubectl exec -n training deploy/api -- ldd /usr/bin/python3 | grep libc
        libc.so.6 => /lib/x86_64-linux-gnu/libc.so.6
```

glibc. Its `RES_TIMEOUT` is 5 s (`resolv/resolv.h`), which matches the observed spike exactly. If this had been `libc.musl-*.so`, a 5.0 s spike would be the *wrong* number — musl retransmits at `timeout/attempts = 2.5 s` — and I would go looking for a different mechanism.

**Step 3 — confirm the drop at the kernel, on the node where it happens.**

```bash
$ NODE=$(kubectl get pod -n training -l app=api -o jsonpath='{.items[0].spec.nodeName}')
$ kubectl debug node/$NODE -it --image=nicolaka/netshoot -- \
    sh -c 'conntrack -S | awk -F"insert_failed=" "{print \$2}" | awk "{s+=\$1} END {print s}"; \
           sleep 60; \
           conntrack -S | awk -F"insert_failed=" "{print \$2}" | awk "{s+=\$1} END {print s}"'
2914
3061
```

**147 failed inserts in 60 seconds on this node.** Diffing matters — the absolute number is since boot and tells you nothing. Sanity-check the arithmetic: the pod is doing roughly 8 lookups/s × 2 queries, ~2 % of which race, which is order-of-magnitude consistent with 2–3 failures/s across all pods on a busy node.

**Step 4 — count the amplification, because it is half the load.**

```bash
$ kubectl exec -n training deploy/api -- cat /etc/resolv.conf
nameserver 10.96.0.10
search training.svc.cluster.local svc.cluster.local cluster.local eu-west-1.compute.internal
options ndots:5

# Watch what one lookup of an EXTERNAL name actually costs:
$ kubectl exec -n training deploy/api -- sh -c \
    'timeout 3 tcpdump -i any -nn -c 20 "udp port 53" 2>/dev/null | grep -c "A?"' &
$ kubectl exec -n training deploy/api -- getent hosts s3.eu-west-1.amazonaws.com >/dev/null
10
```

Ten wire queries for one name. `(4 search entries + 1 absolute) × 2 address families = 10`, exactly as §3 predicts. Fleet-wide this is the real CoreDNS load:

```
   16,000 pods × 2 new conn/s × 10 queries = 320,000 q/s
   CoreDNS running 6 replicas → 53,000 q/s per replica.
   That is well above what a replica sustains comfortably, which is why
   coredns_dns_request_duration_seconds p99 is also elevated — a SECOND,
   independent problem hiding behind the first.
```

**Step 5 — apply the two fixes and measure each separately.** Do not bundle them; you want to know which one did what.

```bash
# Fix A — NodeLocal DNSCache (structural: removes DNAT for the hot path).
kubedns=$(kubectl get svc kube-dns -n kube-system -o jsonpath='{.spec.clusterIP}')
sed -e "s/__PILLAR__LOCAL__DNS__/169.254.20.10/g" \
    -e "s/__PILLAR__DNS__DOMAIN__/cluster.local/g" \
    -e "s/__PILLAR__DNS__SERVER__/$kubedns/g" \
    nodelocaldns.yaml | kubectl apply -f -

# Re-run the 500-lookup histogram from Step 1.
<10ms:499  <100ms:1  <1s:0  <4.9s:0  >=4.9s:0
```

```yaml
# Fix B — kill the amplification for this workload.
# In the Deployment's pod spec:
spec:
  dnsConfig:
    options:
      - name: ndots
        value: "2"
      - name: single-request-reopen     # glibc-only belt-and-braces
```

```bash
# Re-run the external-name query count from Step 4.
$ kubectl exec -n training deploy/api -- sh -c \
    'timeout 3 tcpdump -i any -nn -c 20 "udp port 53" 2>/dev/null | grep -c "A?"'
2
```

**Step 6 — report the result in the units the business uses.**

```
   Tail latency:   p99  5.01 s  →  0.006 s          (the 5 s mode is gone)
   Stall rate:     2.2 % of lookups → 0 in 500      (measure longer to bound it)
   CoreDNS load:   320,000 q/s → ~32,000 q/s        (10× from local cache hits
                                                     and 5× from ndots, overlapping)
   CoreDNS cost:   6 replicas were insufficient; 3 are now comfortable
   Conntrack:      insert_failed on the node 147/min → 0/min
   Residual risk:  external-name cache MISSES still forward over UDP from the
                   node-local agent (the `.:53` block has no force_tcp), so the
                   race is reduced, not provably eliminated. Keep the
                   insert_failed alert.
```

## Practice

<feeds [packet-path teardown + debug runbook](../practice/packet-path-and-debug/README.md)>

**Task A — the resolver comparison matrix.** Build three pods from three base images — `debian:stable-slim` (glibc), `alpine:3` (musl), and a tiny Go binary on `gcr.io/distroless/static` — in the same namespace on the same node. From each, resolve the same relative external name 500 times and histogram the durations. Capture with `tcpdump -i any -nn 'udp port 53'` from a debug container on the node and count queries per lookup. Produce a table with: number of wire queries per lookup, source-port behaviour (one socket or two), the timeout duration observed on a stall, and which `options` each honours. **The deliverable is the table, because it is the thing you will reach for in an incident.**

**Task B — reproduce the stall on purpose.** On a node under load, run parallel `getent hosts` loops from several pods to raise the collision probability, and correlate the ≥ 4.9 s events with the `insert_failed` delta from `conntrack -S` sampled every 10 s. Plot both series on one time axis. Acceptance: a visible correlation, and a stated stall rate with a confidence interval, not a single anecdote.

**Task C — prove the ndots arithmetic.** For five names spanning 0, 1, 3, 4, and 5 dots — plus one with a trailing dot — predict the wire-query count from `(N_search + 1) × 2` and the `dots >= ndots` rule *before* capturing, then capture and compare. Explain any discrepancy (the usual cause is an intermediate cache answering, or a name where the as-is attempt succeeds and short-circuits the walk).

**Task D — the before/after.** Deploy NodeLocal DNSCache and re-run tasks B and C. Record: p99 lookup latency, stall rate, queries reaching CoreDNS (`coredns_dns_requests_total` rate), and node `insert_failed` rate. State explicitly which of the two fixes is responsible for which improvement.

**Acceptance criteria**

- [ ] A three-resolver comparison table backed by captured evidence, including the 5.0 s vs 2.5 s timing distinction.
- [ ] A latency histogram showing the bimodal signature, with the stall rate quantified.
- [ ] A `conntrack -S` `insert_failed` delta correlated in time with the stalls.
- [ ] The ndots query-count prediction matched against a packet capture for at least five names.
- [ ] A before/after table for NodeLocal DNSCache with CoreDNS QPS, p99, and stall rate.
- [ ] A written CoreDNS replica-count calculation using the `R_conn × (N+1) × 2 × (1 − h)` formula with your cluster's real numbers.
- [ ] Three runbook entries: the 5 s stall, the ndots/NXDOMAIN storm, and the SERVFAIL storm — each with symptom, first command, expected output, and fix.

## Common pitfalls

- **"The 5 s stall means DNS is down."** DNS resolved correctly, twice. One dropped UDP query plus a fixed 5-second retransmit timer produced the wait. Restarting CoreDNS cannot fix a kernel-level conntrack insert clash; only removing the DNAT (NodeLocal DNSCache, eBPF datapath) or the parallel-single-socket send (`single-request*`) can.
- **Applying `single-request-reopen` to an Alpine image.** musl's option parser recognises only `ndots`, `attempts`, and `timeout` (`src/network/resolvconf.c`). The option is silently ignored. Worse, the *expected* stall on musl is 2.5 s (`timeout / attempts`), so if you are seeing 5.0 s on Alpine you are chasing the wrong mechanism entirely.
- **"Lower `ndots` to 1 everywhere."** With `ndots:1`, `payments.billing` (1 dot) is tried absolute-first and fails, because that absolute name does not exist. `ndots` is a per-workload decision that depends on the naming convention the calling code uses. `ndots:2` is usually the safe compromise for code that uses `<svc>.<ns>`; anything lower requires FQDNs everywhere.
- **"NodeLocal DNSCache eliminates the race."** It removes DNAT from the hot path, and the shipped Corefile forces TCP for the cluster-domain upstream hop. But the `.:53` block that handles external names forwards over **UDP** with no `force_tcp`. Cache misses for external names still traverse a UDP path. Keep the `insert_failed` alert.
- **"SERVFAIL is never cached."** CoreDNS caches SERVFAIL for **5 seconds** by default (`servfail DURATION`, max 5 minutes, 0 to disable). The damper is short, not absent — which changes what you should expect on the graph during an upstream fault.
- **Setting a long negative TTL to survive an NXDOMAIN storm.** You have just made your negative TTL the floor on how long a newly created Service takes to become resolvable for pods that asked early. Negative TTL is rollout latency.
- **"A headless Service's DNS is instant and free."** It goes through the identical CoreDNS path with the identical ndots expansion. What changes is only the *content* of the answer — an RRset of pod IPs instead of one ClusterIP — which moves load balancing and staleness into the client, where you can no longer fix it centrally.
- **Trusting `dig` to represent the application.** `dig` uses its own resolver, sends one query type, and does not do the search-list walk the way `getaddrinfo` does. Reproduce with `getent hosts` (which goes through NSS and therefore through the real libc path) or with the application's own runtime, and capture packets to see the truth.

## Self-check

**1. A pod's outbound-call latency histogram is cleanly bimodal: a dense mode at ~2 ms and a razor-thin spike at exactly 5.0 s covering 1 % of calls. DNS or application? What is the mechanism, and what is your first command?**

DNS, and specifically the conntrack DNAT insert race. The tell is the *shape*: a fixed timer produces a spike with nothing between the modes, whereas a loaded server produces a smear. 5.0 s is glibc's `RES_TIMEOUT` (`resolv/resolv.h`), so first confirm the image is glibc-based — on musl the corresponding number would be 2.5 s (`timeout / attempts`). Mechanism: glibc sends A and AAAA back to back from one UDP socket; both are NEW to conntrack, both get DNATed from the kube-dns ClusterIP, and one loses the confirmation race and is dropped silently, so the resolver waits out its full retransmit timer. First command: `conntrack -S` twice sixty seconds apart on the pod's node, diffing `insert_failed` and `drop`. Fix ladder: NodeLocal DNSCache (removes the DNAT), then `single-request-reopen` for glibc workloads, then an eBPF dataplane that never DNATs.

**2. Resolving `api.internal.example.com` from inside a pod issues 8+ queries, most NXDOMAIN. Why, and what is the one-character fix?**

`ndots:5`. The name has 3 dots; `3 < 5`, so every resolver treats it as relative and walks the search list before trying it absolute. With a 3-entry search list that is `(3 + 1) × 2 = 8` wire queries; with a 4-entry list including a VPC-injected domain it is 10. One-character fix: a trailing dot — `api.internal.example.com.` — which every resolver treats as absolute and never searches (glibc: `if (dots >= statp->ndots || trailing_dot)` then query as-is and, for a trailing dot, return unconditionally). That drops it to 2 queries, A and AAAA. Systemic fixes: `dnsConfig` with a lower `ndots` per workload, or CoreDNS `autopath` to do the walk server-side — noting that `autopath` needs `pods verified`, mis-attributes namespace when a pod IP is recycled faster than the API watch propagates, and saves nothing when the final answer is negative anyway.

**3. Why is DNS TTL-based discovery a bad primitive for fast failover, and what do you use instead?**

Because the staleness window is bounded only by the TTL and you do not control the clients' caching. After an endpoint is removed, clients holding the record keep dialing it for up to the full TTL — and often longer, because application runtimes cache on top of DNS (the JVM's `networkaddress.cache.ttl` being the notorious case). You cannot fix it by shortening the TTL: at 16,000 pods and a 1 s TTL you generate `16,000 × (N+1) × 2` queries per second of pure overhead, and the window is still up to a second plus whatever the client library adds. The alternative is an indirection whose backing set changes synchronously: a VIP or L4 load balancer where the dataplane rule set is updated when the endpoint is removed (no client re-resolves anything), or a watch-based control plane (EndpointSlice informers, xDS, etcd/Consul watch) that pushes the change in milliseconds. The cost of push is per-client state on the server and a thundering-herd risk on control-plane reconnect.

**4. Your upstream resolver starts intermittently returning SERVFAIL. Ten minutes later, load against that upstream has *increased*. Why, and what is the fix?**

Two amplifiers stack. First, a SERVFAIL is only briefly cached — CoreDNS's default is **5 seconds** (`servfail DURATION`, capped at 5 minutes), which damps but does not stop the re-query rate; every client that would have been served from a healthy 30-second cache entry now re-queries roughly six times as often. Second, clients themselves usually retry on SERVFAIL, adding their own multiplier. There is also a specific CoreDNS behaviour to know: when *all* upstreams are marked unhealthy, `forward` assumes health checking has itself failed and sprays queries to a random upstream anyway, incrementing `coredns_forward_healthcheck_broken_total` — so a fully dead upstream still receives full load. The fix is not more caching: it is `max_fails` and health checking doing their job (0.5 s probe interval, 2 consecutive failures to mark down by default), a second upstream in the `forward` list so there is somewhere healthy to fail over to, `failfast_all_unhealthy_upstreams` if you would rather SERVFAIL immediately than spray, and client-side retry backoff. Widening `servfail` to 30 s is a legitimate temporary damper during a known-bad upstream.

**5. You are sizing CoreDNS for a new 800-node cluster: 32,000 pods, an estimated 1.5 new outbound connections per pod per second, a 4-entry search list, and no node-local cache yet. How many replicas, and what is the single highest-leverage change?**

```
   R_conn = 32,000 × 1.5                      = 48,000 conn/s
   amplification = (4 + 1) × 2                = 10×
   QPS = 48,000 × 10                          = 480,000 q/s
   at ~10,000 q/s per replica (measure yours)  → 48 replicas
```

Forty-eight replicas is a smell, not a plan: it means DNS is now a large, churning workload with its own scheduling and rollout risk. The highest-leverage change is **NodeLocal DNSCache**, because it attacks the multiplier rather than the divisor — at a 90 % local hit rate the CoreDNS-facing load drops to 48,000 q/s and ~5 replicas, and it simultaneously removes the DNAT that causes the 5-second stalls and the UDP conntrack entries that linger for `nf_conntrack_udp_timeout` (30 s). The second-highest-leverage change costs no infrastructure at all: HTTP keep-alive and connection pooling in the applications, which reduces `R_conn` at the source and reduces conntrack pressure at the same time.

**6. Why does the shipped NodeLocal DNSCache Corefile use `force_tcp` for the cluster-domain block but not for the `.:53` block, and what does that mean operationally?**

`force_tcp` on the cluster-domain block sends cache misses to the kube-dns Service over TCP. That matters for two reasons stated in the Kubernetes docs: a TCP conntrack entry is torn down when the connection closes, whereas a UDP entry must age out over `nf_conntrack_udp_timeout` (30 s by default), so TCP does not accumulate table entries; and TCP has a handshake and retransmission, so a single dropped packet does not cost a full 5-second resolver timeout. The `.:53` block handles *external* names and forwards to the node's own upstream resolvers, which are frequently a cloud VPC resolver that may not accept or may throttle TCP — so the shipped config leaves it on UDP. Operationally: NodeLocal DNSCache converts your cluster-name DNS to a TCP-backed, conntrack-light path, but external-name cache misses still take a UDP path, so you keep the `insert_failed` alert and you should still fully-qualify external names to keep those misses rare.

## Connections & what's next

This lesson connects directly back to lesson 01's backpressure framing — the 5-second stall is a UDP-specific instance of a kernel-datapath drop, with its own counter (`conntrack -S`), diagnosed by the same discipline of naming the exact stage — and to `modules/01b-linux-internals/lessons/07-networking-datapath-conntrack.md`, whose "confirmation happens last" fact is the whole reason the race exists.

It sets up lesson 03: DNS's poll-and-TTL discovery is one half of the push-vs-poll axis, and load balancers are where you meet the other half — active health checks, connection draining, and synchronous endpoint removal solving the same "how does a client learn a backend is gone" problem with a different mechanism and a different cost profile. Lesson 05 returns to CoreDNS as one more workload behind a Service VIP, subject to the iptables/IPVS/eBPF mechanics covered there, and shows exactly how the socket-level translation in "rung 3" of the fix ladder works.

## References & further reading

**Primary sources — read directly for this lesson**

1. glibc `resolv/resolv.h`, `resolv/res_init.c`, `resolv/res_send.c`, `resolv/res_query.c` (https://github.com/bminor/glibc/tree/master/resolv) — `RES_TIMEOUT = 5` seconds and `RES_DFLRETRY = 2` (the source of the 5-second stall); default `ndots = 1`; `RES_MAXNDOTS = 15`; the `RES_SNGLKUP`/`RES_SNGLKUPREOP` flags behind `options single-request` / `single-request-reopen`; the retransmit computation `seconds = statp->retrans << ns` with division by nameserver count; the adaptive self-switch to single-request after a timeout that follows a partial answer; and the exact search-list rule `if (dots >= statp->ndots || trailing_dot)`.
2. musl `src/network/resolvconf.c`, `res_msend.c`, `lookup_name.c` (read via the `kraj/musl` GitHub mirror, `https://github.com/kraj/musl`) — defaults `ndots = 1`, `timeout = 5`, `attempts = 2`; the **`retry_interval = timeout / attempts` = 2.5 s** retransmit that distinguishes musl's stall signature from glibc's; all queries sent from a single `SOCK_DGRAM` socket; and the fact that only `ndots`, `attempts`, and `timeout` are parsed from `options` — there is no `single-request` support. *(The canonical musl repository at `git.musl-libc.org` was not reachable from this environment; the mirror was used and the file contents are byte-identical to upstream `src/network/`.)*
3. Go `src/net/dnsconfig_unix.go` and `src/net/dnsclient_unix.go` (https://github.com/golang/go) — defaults `ndots: 1`, `timeout: 5 * time.Second`, `attempts: 2`; `nameList()` implementing the same `dots >= ndots` and trailing-dot rules; A and AAAA dispatched as separate goroutines each with their own `tryOneName` (hence separate sockets); and support for `options single-request` (`conf.singleRequest`) and `use-vc` (`conf.useTCP`).
4. `pkg/kubelet/network/dns/dns.go`, Kubernetes master (https://github.com/kubernetes/kubernetes) — `defaultDNSOptions = []string{"ndots:5"}` and the cluster search-list construction `<ns>.svc.<domain>`, `svc.<domain>`, `<domain>`.
5. `pkg/apis/core/validation/validation.go`, Kubernetes master — `MaxDNSNameservers = 3`, `MaxDNSSearchPaths = 32`, `MaxDNSSearchListChars = 2048`.
6. `cluster/addons/dns/coredns/coredns.yaml.base`, Kubernetes master — the default Corefile reproduced and annotated in §8, including `health { lameduck 5s }`, `kubernetes … { pods insecure; fallthrough; ttl 30 }`, `forward . /etc/resolv.conf { max_concurrent 1000 }`, `cache 30`, `loop`, `reload`, `loadbalance`.
7. `cluster/addons/dns/nodelocaldns/nodelocaldns.yaml`, Kubernetes master — the NodeLocal DNSCache Corefile, including the `force_tcp` on the cluster-domain upstream hop, the `cache { success 9984 30; denial 9984 5 }` split, and the fact that the `.:53` external-name block carries **no** `force_tcp`.
8. CoreDNS `plugin/cache/README.md` (https://github.com/coredns/coredns) — max TTL 3600 s for NOERROR and 1800 s for denials, minimum TTL 5 s, 256 shards × 39 items = 9984 entries per cache type, per-shard eviction, `prefetch` defaults (1 m duration, 10 %), `serve_stale` default 1 h, and — **correcting a common claim and the previous version of this lesson** — that SERVFAIL responses **are** cached, for 5 seconds by default, tunable via `servfail DURATION` up to a 5-minute maximum.
9. CoreDNS `plugin/forward/README.md` — in-band health checking at a **0.5 s** interval using a `. IN NS` query, `max_fails = 2`, `read_timeout = 2 s`, `expire = 10 s`, `policy = random` by default, a 15-upstream limit, `max_concurrent` costing ~2 KB per in-flight query, the all-upstreams-down spray behaviour with `coredns_forward_healthcheck_broken_total`, and `failfast_all_unhealthy_upstreams`.
10. CoreDNS `plugin/autopath/README.md` — the server-side search-path mechanism, the `@kubernetes` integration, and the documented bugs: wrong-namespace attribution when a pod IP is recycled ahead of the API watch, Windows-node incompatibility, and the negative-answer case where the client re-walks every path anyway.
11. CoreDNS `plugin/kubernetes/README.md` and `plugin/loop/README.md` — the `ttl` default of 5 s (max 3600), `pods insecure|verified|disabled`, and the loop plugin's random-`HINFO`-probe startup check that halts CoreDNS on a detected forwarding loop.
12. CoreDNS `plugin/metrics/README.md` and `plugin/cache/README.md` metrics sections — the metric names used in §8: `coredns_dns_requests_total{server,zone,view,proto,family,type}`, `coredns_dns_responses_total{…,rcode,plugin}`, `coredns_dns_request_duration_seconds`, `coredns_cache_entries`, `coredns_cache_hits_total`, `coredns_cache_evictions_total`, `coredns_forward_max_concurrent_rejects_total`, `coredns_forward_healthcheck_broken_total`.
13. `kubernetes/dns`, `docs/specification.md` (https://github.com/kubernetes/dns/blob/master/docs/specification.md) — the normative DNS-based service discovery schema used in §9: A/AAAA, SRV (`_<port>._<proto>.<service>.<ns>.svc.<zone>.`), and PTR forms for ClusterIP Services; the headless-Service RRset form; and the `dns-version.<zone>. IN TXT` schema record.
14. `kubernetes/website`, `content/en/docs/tasks/administer-cluster/nodelocaldns.md` — the motivation section quoted in §6 and §11: avoiding iptables DNAT and connection tracking to reduce conntrack races (citing kubernetes/kubernetes#56903), TCP conntrack entries being removed on close versus UDP entries timing out at `nf_conntrack_udp_timeout` (30 s default), and the link-local `169.254.0.0/16` / ULA `fd00::/8` recommendation for the local listen address.

**Sources named but not fetched in this pass — do not treat the wording as verified**

- RFC 2308 (negative caching of DNS queries) and RFC 8020 (NXDOMAIN means there is nothing underneath). `www.rfc-editor.org`, `ietf.org`, and `datatracker.ietf.org` are blocked by this environment's egress policy, so neither document was read for this rewrite. The negative-caching *behaviour* described in §7 is taken from the CoreDNS `cache` plugin's documented defaults rather than from the RFC text.
- kubernetes/kubernetes issue #56903 itself. The GitHub API is not reachable for this repository from this session; the issue is cited here only as the reference the Kubernetes NodeLocal DNSCache task page gives for "conntrack races," and that task page *was* read directly.
- Weave Works, "Racy conntrack and DNS lookup timeouts" (https://www.weave.works/blog/racy-conntrack-and-dns-lookup-timeouts) — the widely cited write-up of this race. Not reachable from this environment; the mechanism in §5 is reconstructed from glibc's source, the kernel's confirmation ordering (via `modules/01b-linux-internals/lessons/07-networking-datapath-conntrack.md`), and the Kubernetes documentation's own account.
- Datadog, "It's always DNS … except when it's not" (https://www.datadoghq.com/blog/engineering/grpc-dns-and-load-balancing-incident/) — an incident where a DNS-shaped symptom turned out to be conntrack and reverse-path-filter behaviour above the node level. Not reachable from this environment; retained as an optional pointer only, and no figures from it are asserted in this lesson.
