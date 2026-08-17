---
lesson: "01.7"
title: "Standard library fluency: net/http, json, slog, flags, and the Prometheus exporter idiom"
module: "01"
concept: "Standard library fluency: net/http, json, slog, flags, and the Prometheus exporter idiom"
status: not-started
est_time: "12h"
prev: "06-modules-and-layout.md"
next: "08-reading-unfamiliar-go.md"
artifacts: []
sources: 13
---

# 01.7 · Standard library fluency: net/http, json, slog, flags, and the Prometheus exporter idiom

> **Concept.** The stdlib (`net/http`, `encoding/json`, `log/slog`, `flag`, `time`, `io`, `os`) plus `prometheus/client_golang` is the entire raw material of an exporter or CLI — learn its idioms and gotchas cold.
>
> Module: [🐹 01 — Go for infrastructure engineers](../README.md) · Deliverable: [`gpu-cost-exporter`](../practice/gpu-cost-exporter/README.md)

## Where this fits

Lesson 6 gave you the *container* — `go.mod`, `internal/`, and the layout a controller repo is judged on before anyone reads a line of logic. This lesson gives you what goes *inside* that container: the four stdlib moves (parse flags, log structured, serve HTTP, decode JSON) plus the one third-party idiom you'll use everywhere — `prometheus/client_golang`. By the end you'll have written the actual scrape-and-serve core of `gpu-cost-exporter`, not a toy. That matters immediately, because lesson 8, **Reading unfamiliar Go**, asks you to go read the *real* dcgm-exporter and node_exporter source that this lesson's exporter idiom is modeled on — you can't read those critically until you've built the pattern yourself and felt where it's easy to get wrong.

## Why this matters

Every exporter you operate — node-exporter, kube-state-metrics, dcgm-exporter — is the same four moves: parse flags, log structured JSON, serve HTTP, register metrics. In Python you'd reach for `argparse` + `logging` + `flask` + `prometheus_client` and pull in a web framework. In Go the standard library *is* the framework, and it ships in the binary.

Getting these idioms wrong is how real exporters leak goroutines, hang on a dead upstream, or DoS your own control plane. Two failure modes dominate, and both are invisible in review because both compile and both pass a happy-path test:

1. **A client with no timeout.** `http.DefaultClient.Timeout` is the zero value, which `net/http` documents as "no timeout." A hung upstream parks a goroutine and a TCP connection forever. Your ticker fires again 30 seconds later and parks another. Nothing errors, nothing logs, memory climbs, and the exporter OOMs hours later with a stack dump full of `net.(*netFD).Read`.
2. **A response body that is closed but never drained.** Go returns a connection to the idle pool *only* if the body was read to `io.EOF`. Close early and the connection is destroyed instead of reused, so every scrape pays a fresh TCP (and possibly TLS) handshake. On a DaemonSet across 500 GPU nodes that is 500 × handshake-per-scrape of avoidable load on whatever you're scraping.

You will fix both in this lesson, and you'll see the exact line of stdlib source that makes each true. This is also a literal interview signal: CoreWeave- and NVIDIA-class JDs ask for "backend services and Go programs deployed to Kubernetes," and the take-home format for these roles is almost always "write a small tested service" — precisely this shape of code, timeouts and all.

## What's new here (calibration)

Per the module README's skip-list, we do **not** re-teach: what HTTP is, what JSON is, what a CLI flag is, or how to stand up a web framework (Flask/Express-equivalents) — you already know those shapes from Python. We also don't re-derive `argparse`/`logging` basics; you know *why* you'd want flags and structured logs, so we go straight to the Go-specific mechanics and gotchas.

What's genuinely new here, calibrated to staff-level GPU-fleet depth:

- **Every clock in an HTTP request**, named, with the segment of the timeline it bounds and what its zero value means. There are nine of them across `Client`, `Transport`, and `Server`, and "set a timeout" is not one decision, it's nine.
- **The connection pool as a data structure** — the idle list, the LRU, `MaxIdleConnsPerHost = 2`, and the exact condition in `transport.go` that decides whether your connection is reused or thrown away.
- **`encoding/json` as a strict-by-omission decoder**: silently-dropped unknown fields, case-insensitive key matching, `float64`-by-default numbers, and the difference between `omitempty` and Go 1.24's `omitzero`.
- The difference between `promauto`'s eager, `Set`-on-a-timer convenience wrappers and a **`Desc`-based custom `Collector`** that computes values lazily per-scrape — the pattern real exporters like dcgm-exporter and node_exporter actually use.
- **Concurrent collection inside a single scrape** — the bounded-fan-out pattern from Lesson 4 (`errgroup.SetLimit`) showing up *inside* the exporter idiom itself, exactly as node_exporter implements it.
- Full `http.Server` hardening and `slog` redaction (`ReplaceAttr`) — the kind of finding a real `gosec`/security review leaves on a PR, not textbook material.

Everything below is checked against the Go 1.24 standard library source (`go1.24.7`) and `prometheus/client_golang v1.24.1`. Where a default is a named constant in the source, the constant is given.

## Core concepts

### From Python to Go: the delta

| You did in Python | You do in Go | The delta that bites |
|---|---|---|
| `requests.get(url)` (default: **no timeout**) | `&http.Client{Timeout: 5*time.Second}` | Go's *default* client also has no timeout, but you construct it explicitly, so there's no excuse. `http.Get` uses `DefaultClient` = unbounded. |
| `requests.Session()` pools connections for you | `http.Transport` pools connections, keyed by (proxy, scheme, host) | The pool is on the `Transport`, not the `Client`. Constructing a new `Client` per call with a new `Transport` throws the pool away every time. |
| `resp.json()`, body consumed for you | `defer resp.Body.Close()` — and drain it | Close alone is not enough for connection reuse. Go decides reuse on "did the body reach EOF?", not on "was Close called?" |
| `json.loads` → `dict` | `json.Unmarshal` → struct with tags | No dict-of-anything by default. You declare the shape. Unknown fields are silently dropped, not errored. |
| `logging.info("cost=%s", c)` | `slog.Info("scrape done", "cost", c)` | Key/value pairs, not format strings. Machine-parseable at the source. |
| `argparse` | `flag` (stdlib) or `cobra` (subcommands) | `flag` is tiny and built in; `cobra` for `verb noun` CLIs like `kubectl`. |
| `prometheus_client.Gauge(...).set(x)` | `promauto.NewGaugeVec(...).WithLabelValues(...).Set(x)` | Metrics are registered into a *registry* at construction; label cardinality is your responsibility (and your bill). |
| `flask` route | `http.ServeMux` + `http.HandlerFunc` | No framework. `promhttp.Handler()` is just an `http.Handler` you mount. |

The mental shift: Python gives you dynamic convenience and you bolt on rigor; Go gives you rigor and you can't opt out. Structs-with-tags replace duck typing, explicit timeouts replace hopeful defaults — but only where you actually write them down, because *the Go zero value for a `time.Duration` timeout field is universally "no timeout."* That is the single most important sentence in this lesson. Go does not have safe defaults for network deadlines; it has *absent* defaults, and absence means infinity.

### `net/http` client: the request lifecycle, and every clock that bounds it

#### The problem the timeouts exist to solve

An HTTP request is not one operation. It is a sequence: resolve DNS, open TCP, negotiate TLS, write the request, wait for the server to think, read response headers, read the response body. Each of those steps can block independently and for different reasons. A single "request timeout" is a blunt instrument: if you set it to 5 s, a healthy 200 ms upstream serving a 4 GB body fails just as surely as a black-holed one.

So `net/http` gives you a stack of clocks at different granularities. You do not need all of them, but you need to know which segment each one covers, because the segment a clock does *not* cover is where your process will eventually hang.

#### The timeline

```
  client.Do(req)
      │
      ├─ [1] get a connection from the pool ─── or dial a new one ─────────────┐
      │                                                                        │
      │      DNS lookup        TCP connect         TLS handshake               │
      │      ├──────────┤      ├──────────┤        ├────────────┤              │
      │      └─────────── net.Dialer.Timeout ──────┘                           │
      │           (DefaultTransport: 30s — covers DNS + TCP, NOT TLS)          │
      │                                       └── Transport.TLSHandshake ──┘   │
      │                                            Timeout (default 10s)       │
      │                                                                        │
      ├─ [2] write request headers + body ─────────────────────────────────────┤
      │      (if "Expect: 100-continue": wait ≤ Transport.ExpectContinueTimeout)│
      │                                                     default 1s         │
      │                                                                        │
      ├─ [3] wait for the server's response HEADERS ───────────────────────────┤
      │      └────── Transport.ResponseHeaderTimeout (default 0 = ∞) ──────┘   │
      │                                                                        │
      ├─ [4] Do() RETURNS here. resp.Body is still an open stream. ────────────┤
      │                                                                        │
      └─ [5] your code reads resp.Body ... ────────────────────────────────────┘
             (no Transport field bounds this at all)

      ├──────────────────── Client.Timeout ────────────────────────────────────┤
       covers [1]+[2]+[3]+[4]+[5] INCLUDING body reads, INCLUDING all redirects.
       Default 0 = no timeout. The timer keeps running after Do() returns and
       will interrupt an in-progress Body.Read with a timeout error.

      ├──────────────────── ctx deadline (NewRequestWithContext) ──────────────┤
       covers the same span as Client.Timeout, but is cancellable, composable,
       and propagates to anything else selecting on ctx.Done().
```

Read that diagram twice. The two facts most people get wrong are both in it:

- **`Client.Timeout` covers the body read**, and its timer *does not stop when `Do` returns*. This is documented on the field: "The timer remains running after Get, Head, Post, or Do return and will interrupt reading of the Response.Body." So a 5 s `Client.Timeout` on a scrape that streams a 200 MB response will fail at 5 s even though headers arrived in 20 ms. If you stream large bodies, use a generous `Client.Timeout` (or none) plus a tight `Transport.ResponseHeaderTimeout`, not the other way around.
- **Nothing in the `Transport` bounds step [5].** `ResponseHeaderTimeout` explicitly stops at headers — the doc comment says "This time does not include the time to read the response body." A server that sends `200 OK` and then trickles one byte per minute forever is not stopped by any `Transport` field. Only `Client.Timeout` or a `ctx` deadline stops it.

#### `http.Transport` — the field table

This is the whole hardening surface. Defaults are those of `http.DefaultTransport`, which is what you get when `Client.Transport` is nil.

| Field | Type | `DefaultTransport` value | What zero means |
|---|---|---|---|
| `Proxy` | `func(*Request) (*url.URL, error)` | `ProxyFromEnvironment` | nil = no proxy, ever. Setting a custom `Transport` and forgetting this is how `HTTPS_PROXY` silently stops working. |
| `DialContext` | `func(ctx, network, addr)` | `net.Dialer{Timeout: 30s, KeepAlive: 30s}` | nil = a plain dialer with **no dial timeout** — DNS + TCP connect can hang indefinitely. |
| `TLSHandshakeTimeout` | `time.Duration` | `10 * time.Second` | 0 = no timeout on the TLS handshake. |
| `ExpectContinueTimeout` | `time.Duration` | `1 * time.Second` | 0 = don't wait for `100 Continue`; send the body immediately. |
| `ResponseHeaderTimeout` | `time.Duration` | **0 (unset)** | 0 = wait forever for the server's response headers. |
| `IdleConnTimeout` | `time.Duration` | `90 * time.Second` | 0 = idle keep-alive connections are never closed by age. |
| `MaxIdleConns` | `int` | `100` | 0 = no limit on total idle connections across all hosts. |
| `MaxIdleConnsPerHost` | `int` | **0 → falls back to `DefaultMaxIdleConnsPerHost = 2`** | Not "unlimited". Two. This one bites; see below. |
| `MaxConnsPerHost` | `int` | **0 (unset)** | 0 = no cap on total (dialing + active + idle) connections per host. Non-zero makes excess dials *block*. |
| `DisableKeepAlives` | `bool` | `false` | false = keep-alive on. Setting true forces a new connection per request. |
| `DisableCompression` | `bool` | `false` | false = the Transport adds `Accept-Encoding: gzip` and transparently decodes. |
| `ForceAttemptHTTP2` | `bool` | `true` | false + any custom dialer or `TLSClientConfig` = HTTP/2 silently disabled. |
| `MaxResponseHeaderBytes` | `int64` | 0 (a package default applies) | 0 = an internal default limit, not "unlimited". |
| `WriteBufferSize` / `ReadBufferSize` | `int` | 0 → 4 KiB each | 0 = 4096 bytes (`4 << 10` in `transport.go`). |

The two rows that decide production behaviour:

**`ResponseHeaderTimeout` is unset by default.** This is the clock you actually want for a scrape: it bounds "the upstream accepted my connection and then went quiet," which is the single most common shape of a wedged dependency. A GPU cost API that is up, listening, TLS-terminating, and then blocked on a slow database will hold your connection forever with the default transport. Set it.

**`MaxIdleConnsPerHost` defaults to 2.** Here is the arithmetic that makes it matter. Suppose your exporter fans out 16 concurrent requests to one upstream host per scrape cycle (Lesson 4's bounded `errgroup`). All 16 dial. When they finish, `tryPutIdleConn` runs for each; the first 2 are stored in `t.idleConn[key]`, and the remaining 14 hit `if len(idles) >= t.maxIdleConnsPerHost() { return errTooManyIdleHost }` and are **closed**. Next cycle, 14 fresh TCP connections and 14 fresh TLS handshakes. At a 30 s interval that is 14 × 2 = 28 handshakes/minute per exporter instance; on a 500-node GPU DaemonSet that is 14,000 avoidable TLS handshakes per minute pointed at one API. Fix: set `MaxIdleConnsPerHost` to at least your fan-out width.

#### Client.Timeout vs. context: pick both, for different reasons

```go
client := &http.Client{
    Timeout:   30 * time.Second, // hard ceiling, includes body read + redirects
    Transport: tr,
}

ctx, cancel := context.WithTimeout(parent, 5*time.Second)
defer cancel()
req, err := http.NewRequestWithContext(ctx, http.MethodGet, url, nil)
```

They bound the same span, but they are not interchangeable:

- `Client.Timeout` is a **static policy** on the client. Every request through it gets the same ceiling. It cannot be shortened for one call or cancelled early.
- The `ctx` deadline is **per-call and composable**. It inherits from the caller's context, so when the manager cancels on `SIGTERM` (Lesson 4), the in-flight HTTP request unwinds immediately instead of waiting out its timeout. It also propagates: the same `ctx` you pass to `NewRequestWithContext` is the one your `select` loops and downstream calls watch.

The rule: **`ctx` is the deadline you reason about; `Client.Timeout` is the seatbelt for the code path that forgot one.** In a controller or an exporter you thread `ctx` everywhere, so `ctx` does the real work — but you still set `Client.Timeout` because a future contributor will add a call site that passes `context.Background()`.

One asymmetry worth knowing: a `ctx` cancellation surfaces as an error wrapping `context.DeadlineExceeded` or `context.Canceled`, which `errors.Is` can classify (Lesson 2 — retryable vs terminal). `Client.Timeout` surfaces as a `*url.Error` whose `Timeout()` method returns true; since Go 1.23 it also wraps `context.DeadlineExceeded`, but classifying on `errors.Is(err, context.DeadlineExceeded)` is only reliable if you own both ends. For classification code, prefer driving everything from `ctx`.

### The connection pool, and where a leaked body blocks it

#### The problem

TCP + TLS setup is expensive: a handshake is 1–2 round trips for TCP and 1–2 more for TLS, so on a 5 ms-RTT intra-cluster path you're paying 15–25 ms before a single byte of your request moves. Reusing a warm connection makes that zero. Go's `Transport` therefore keeps an idle-connection pool and hands you a pooled connection when the destination matches.

The subtlety, and the reason this section exists: **Go decides whether a connection is reusable by watching the response body, not by watching your `Close` call.** If your code closes the body before reading it to EOF, the connection cannot be reused, because the transport has no idea how many unread bytes of the previous response are still sitting in the socket — reading the next response off that socket would read garbage.

#### The structure

```
  http.Client                     http.Transport (owns the pool)
  ┌──────────┐                    ┌──────────────────────────────────────────┐
  │ Timeout  │──── RoundTrip ────▶│  idleConn map[connectMethodKey][]*persistConn
  │ Transport│                    │      key = (proxy, scheme, host:port)     │
  └──────────┘                    │      ├─ [0] pconn ── writeLoop goroutine  │
                                  │      └─ [1] pconn ── readLoop  goroutine  │
                                  │      (≤ MaxIdleConnsPerHost = 2 default)  │
                                  │                                           │
                                  │  idleLRU ──── evicts oldest when total    │
                                  │               exceeds MaxIdleConns (100)  │
                                  │  idleTimer ── closes a pconn after        │
                                  │               IdleConnTimeout (90s)       │
                                  └──────────────────────────────────────────┘

  Per-connection readLoop, after handing you resp:

     resp.Body wrapped in bodyEOFSignal
              │
      ┌───────┴────────────────────────────────────┐
      │                                            │
   you read to io.EOF                       you Close() early
      │                                            │
   fn(io.EOF) → waitForBodyRead <- TRUE      earlyCloseFn() → waitForBodyRead <- FALSE
      │                                            │
      ▼                                            ▼
   alive = TRUE && !sawEOF                   alive = FALSE
        && wroteRequest()                          │
        && tryPutIdleConn()                        │
      │                                            │
      ▼                                            ▼
   ┌───────────────────┐                    ┌──────────────────┐
   │ back in idle pool │                    │ connection CLOSED│
   │ reused next call  │                    │ next call redials│
   └───────────────────┘                    └──────────────────┘
```

That `alive = ... && bodyEOF && ...` conjunction is literally the code in `net/http/transport.go`'s `readLoop` (Go 1.24). Every term must be true for the connection to go back in the pool. `bodyEOF` is the term you control.

#### What this means you must write

```go
resp, err := client.Do(req)
if err != nil {
    return fmt.Errorf("scrape %s: %w", url, err)
}
defer func() {
    // Drain up to 64 KiB so the connection is reusable, then close.
    // Both calls are required: the copy makes it poolable, the Close
    // releases it. Close alone leaks the connection; drain alone leaks
    // the file descriptor.
    _, _ = io.Copy(io.Discard, io.LimitReader(resp.Body, 64<<10))
    _ = resp.Body.Close()
}()
```

Three details:

- **`defer` it immediately after the error check**, not after the status check. On a non-2xx response `err` is `nil` and `resp.Body` is a real, open stream — an error-status path that returns before `Close` is the classic leak, and it looks correct in review because there *is* a `defer resp.Body.Close()` five lines further down, just after an early `return`.
- **Bound the drain.** `io.Copy(io.Discard, resp.Body)` on a hostile or broken upstream will happily read gigabytes. `io.LimitReader` caps the cost of politeness. If the body is larger than the limit you've decided the connection isn't worth saving, and closing early is the right call.
- **Nothing here is needed if you fully decoded the body**, *except* that `json.Decoder.Decode` **stops at the end of the first JSON value** and does not read to EOF. A response with a trailing newline after the JSON object — which is what most servers send — leaves one unread byte, `bodyEOF` is false, and the connection is thrown away. This is the highest-value, least-known fact in this section: **decoding JSON is not draining the body.** The `io.Copy` above is what makes a JSON scrape connection-reusing.

#### How you'd see it in production

The symptom is never "connection leak." It is:

- `netstat`/`ss` showing a steady churn of `TIME_WAIT` sockets from your pod, roughly one per request instead of near zero.
- A latency floor that matches your RTT × handshake count and that *doesn't* improve when the upstream gets faster.
- On the upstream: accept-queue and TLS-handshake CPU that scales with your scrape count rather than your byte volume.
- If you *also* forgot `Close`, a slow climb in the process's open FD count (`ls /proc/<pid>/fd | wc -l`) ending in `too many open files`.

### `net/http` server: four clocks, and what zero means for each

A handler is anything implementing `ServeHTTP(http.ResponseWriter, *http.Request)`. `http.HandlerFunc` adapts a plain function. `http.ServeMux` routes; since Go 1.22 its patterns take the form `[METHOD ][HOST]/[PATH]` with `{name}` wildcards, so `mux.HandleFunc("GET /healthz", h)` is valid stdlib routing with no third-party router.

The server's timeout fields have *interlocking* zero-value semantics — this is where most hardening advice is subtly wrong.

| Field | Bounds | Zero value means |
|---|---|---|
| `ReadHeaderTimeout` | Reading the request line + headers. The read deadline is reset once headers are parsed. | **Falls back to `ReadTimeout`.** If `ReadTimeout` is also zero (or negative), no timeout. A *negative* value means "no timeout" explicitly. |
| `ReadTimeout` | The entire request: headers **and** body. | Zero or negative = no timeout. |
| `WriteTimeout` | Writing the response. Reset whenever a new request's header is read. | Zero or negative = no timeout. |
| `IdleTimeout` | Waiting for the *next* request on a keep-alive connection. | **Falls back to `ReadTimeout`.** If that is zero/negative too, no timeout. |
| `MaxHeaderBytes` | Bytes of request line + header keys/values. Does **not** limit the body. | 0 → `DefaultMaxHeaderBytes = 1 << 20` (1 MiB). |

The fallback chain matters. Setting only `ReadTimeout: 10s` gives you a `ReadHeaderTimeout` of 10 s and an `IdleTimeout` of 10 s for free — which is why "just set ReadTimeout" is not as wrong as it sounds, but is still bad: it means every keep-alive connection is torn down after 10 s idle, defeating Prometheus's own connection reuse across scrapes. Setting all four explicitly is the only way to say what you mean.

```
  Server-side timeline for one keep-alive connection
  ──────────────────────────────────────────────────

  accept
    │
    ├── read request line + headers ────────┤   ReadHeaderTimeout (5s)
    │                                       │
    ├── read body ──────────────────────────┤ ┐
    │                                       │ ├ ReadTimeout (10s) spans BOTH
    ├── handler runs, writes response ──────┤ ┘   headers and body
    │                                       │
    ├───────────────────────────────────────┤   WriteTimeout (10s), reset at
    │                                           each new request's header read
    │
    ├── connection idle, awaiting next req ─┤   IdleTimeout (60s)
    │
    └── next request, or close
```

The attack each one stops:

- **`ReadHeaderTimeout` → slowloris.** An attacker opens N connections and sends one header byte every 20 s. Without this field each connection occupies a goroutine and an FD indefinitely; the process dies of goroutine/FD exhaustion, not of CPU. `gosec` rule G112 flags a `Server` literal with no `ReadHeaderTimeout` for exactly this reason, and it is one of the most common findings on a first Go PR.
- **`ReadTimeout` → slow-body.** Headers arrive instantly, then the body trickles. `ReadHeaderTimeout` has already been satisfied and reset; only `ReadTimeout` bounds this.
- **`WriteTimeout` → slow-reader.** The client stops reading; your handler blocks in `w.Write`, holding whatever it holds (a DB row, a mutex, a GPU handle).
- **`IdleTimeout` → connection hoarding.** Keep-alives that never close accumulate one goroutine and one FD each.

Prefer an explicit `&http.Server{}` over `http.ListenAndServe(addr, mux)` precisely so you can set these; the convenience function constructs a `Server` with every timeout at zero.

**Graceful shutdown** completes the picture. `srv.Shutdown(ctx)` stops accepting new connections, closes idle ones, and waits for active requests to finish or for `ctx` to expire; it returns `http.ErrServerClosed` from `ListenAndServe` in the goroutine that was serving. Without it, `SIGTERM` during a rolling update kills in-flight scrapes and Prometheus records a failed scrape for every restarting pod.

### `encoding/json` — a permissive decoder you have to make strict

#### The tag grammar

Only **exported** (capitalized) fields are marshaled or unmarshaled; the `json` package uses reflection and cannot see unexported fields. The tag format is `json:"name,option,option"`.

```go
type GPUCost struct {
    Namespace string     `json:"namespace"`
    GPUHours  float64    `json:"gpu_hours"`
    USD       *float64   `json:"usd,omitempty"`   // pointer: 0 survives, absent is nil
    Ratio     float64    `json:"ratio,omitzero"`  // Go 1.24: omit if zero (or IsZero())
    Note      string     `json:"-"`               // never marshaled
    Literal   string     `json:"-,"`              // the JSON key is literally "-"
    Big       int64      `json:",string"`         // encoded as a JSON *string*
    Scraped   time.Time  `json:"scraped_at"`      // RFC 3339 via MarshalJSON
}
```

| Option | Effect |
|---|---|
| `omitempty` | Omit if the value is `false`, `0`, a nil pointer, a nil interface, or an array/slice/map/string of length zero. **Note what is not on that list: a zero-valued struct.** `omitempty` on a `time.Time` field does nothing. |
| `omitzero` | (Go 1.24+) Omit if the value is the type's zero value, or if the type has an `IsZero() bool` method that returns true. This is the one that works on `time.Time`. |
| `string` | Encode a numeric/bool/string field as a JSON string. Used for `int64` values that would lose precision in a JavaScript consumer. |
| `-` | Never encode or decode this field. |
| `-,` | The JSON key is the literal `"-"`. |

#### The four behaviours that surprise people

**1. `omitempty` erases a real zero.** A genuinely idle namespace with `usd: 0` disappears from the payload entirely. Downstream, "field absent" and "confirmed zero" become indistinguishable — and for a billing feed those mean very different things (missing data vs. verified no spend). The fix is a `*float64`: `nil` marshals as absent, `&zero` marshals as `0`. Use a pointer whenever "unset" and "zero" are semantically different. Do not reach for `omitzero` here: it has the same problem, it just also handles structs.

**2. Unknown fields are silently dropped.** `Unmarshal` walks the JSON object and, for each key, looks for a matching struct field; keys with no match are skipped with no error. When you're parsing your own exporter's output that's fine and forward-compatible. When you're parsing an upstream billing API it is dangerous: the vendor renames `usd_per_gpu_hour` to `usdPerGpuHour`, your struct still decodes cleanly, and every namespace reports `$0.00`. Guard it:

```go
dec := json.NewDecoder(resp.Body)
dec.DisallowUnknownFields() // now an unexpected key is a decode error
if err := dec.Decode(&rows); err != nil {
    return fmt.Errorf("decode cost payload: %w", err)
}
```

**3. Field matching is case-insensitive.** The decoder prefers an exact match but falls back to a case-insensitive one. So `{"USD_PER_GPU_HOUR": 2.13}` populates a field tagged `json:"usd_per_gpu_hour"`. This is usually a convenience and occasionally a bug: two struct fields whose tags differ only in case will collide, and a hostile payload can populate a field you did not expect it to reach.

**4. Numbers decode to `float64` when the target is `any`.** Unmarshalling into `map[string]any` or `any` turns every JSON number into a `float64`, silently losing integer precision above 2^53 — which matters for nanosecond timestamps and byte counts. `dec.UseNumber()` makes the decoder produce `json.Number` (a string with `.Int64()` / `.Float64()` methods) instead. Better: decode into a typed struct so `int64` stays `int64`.

#### Streaming vs. buffering

```go
// Buffering: reads the entire body into memory first.
b, _ := io.ReadAll(resp.Body)
_ = json.Unmarshal(b, &rows)

// Streaming: decodes as bytes arrive; peak memory is the decoded value,
// not the wire payload.
_ = json.NewDecoder(resp.Body).Decode(&rows)
```

Prefer the decoder for anything you read from a network. On a `/api/v1/pods` response from a busy cluster the wire payload can be tens of megabytes, and `io.ReadAll` allocates all of it, in a slice that grows by repeated doubling — so peak RSS is roughly 2× the payload during the final grow. In a DaemonSet with a 128 MiB memory limit that's an OOMKill on a bad day.

The corresponding gotcha, restated because it's the one that costs you connections: **`Decode` stops after one JSON value.** It leaves the trailing newline (and anything after it) unread, so the body never hits EOF and the connection isn't pooled. Drain after decoding.

On the encoding side, `json.NewEncoder(w)` escapes `<`, `>`, and `&` to `<`, `>`, `&` by default (a legacy HTML-safety measure); `enc.SetEscapeHTML(false)` turns that off. This matters the day a PromQL query or a label value containing `>` shows up mangled in your JSON logs.

### `log/slog` — structured logging, levels, and redaction

#### The architecture

`slog` separates three things Python's `logging` conflates:

- **`Logger`** — the front end you call (`Info`, `Error`, `With`, `WithGroup`).
- **`Record`** — one log event: time, level, message, and a list of `Attr` key/value pairs.
- **`Handler`** — the back end that decides whether to emit a record and how to format it. `slog.NewJSONHandler` and `slog.NewTextHandler` ship in the stdlib; `logr`/`zap` bridges exist for the controller world.

```go
opts := &slog.HandlerOptions{
    Level:     slog.LevelInfo,
    AddSource: false, // true adds "source":{"function":..,"file":..,"line":..}
}
logger := slog.New(slog.NewJSONHandler(os.Stdout, opts))
slog.SetDefault(logger) // package-level slog.Info now routes here

slog.Info("scrape complete",
    "namespace", ns, "gpu_hours", h, "usd", cost,
    "duration_ms", took.Milliseconds())
// {"time":"2026-08-17T09:14:02.113Z","level":"INFO","msg":"scrape complete",
//  "namespace":"team-a","gpu_hours":12.5,"usd":26.6,"duration_ms":41}
```

The four built-in keys are constants: `slog.TimeKey = "time"`, `slog.LevelKey = "level"`, `slog.MessageKey = "msg"`, `slog.SourceKey = "source"`. Built-in levels are `Debug = -4`, `Info = 0`, `Warn = 4`, `Error = 8` — deliberately spaced by 4 so you can define intermediate levels (`Notice = 2`) without renumbering.

#### Runtime-adjustable level

`HandlerOptions.Level` is a `slog.Leveler`, an interface, not a constant. Pass a `*slog.LevelVar` and you can change the level while the process runs — from a `--log-level` flag at startup, or from an HTTP endpoint for live debugging:

```go
var lvl slog.LevelVar // zero value is LevelInfo
lvl.Set(slog.LevelDebug)
logger := slog.New(slog.NewJSONHandler(os.Stdout, &slog.HandlerOptions{Level: &lvl}))
// later, from a handler:
mux.HandleFunc("PUT /debug/level", func(w http.ResponseWriter, r *http.Request) {
    var l slog.Level
    if err := l.UnmarshalText([]byte(r.URL.Query().Get("v"))); err != nil {
        http.Error(w, err.Error(), http.StatusBadRequest)
        return
    }
    lvl.Set(l)
})
```

`LevelVar` is safe for concurrent use, so no mutex is needed. This is worth wiring on day one: turning on debug logging for one pod without a redeploy is the difference between diagnosing an intermittent scrape failure in ten minutes and waiting for the next rollout.

#### Context: `With` vs `WithGroup`

```go
log := logger.With("cluster", "us-east-gpu-1", "exporter", "gpu-cost")
// every record from `log` carries cluster and exporter

sub := logger.WithGroup("upstream").With("host", "cost-api")
sub.Info("call failed", "status", 503)
// {"msg":"call failed","upstream":{"host":"cost-api","status":503}}
```

`With` prepends attributes to every record; `WithGroup` nests subsequent attributes under a key. Use `With` for identity (cluster, node, tenant) and `WithGroup` to avoid key collisions when logging two subsystems' fields in one record.

#### `ReplaceAttr` — redaction that can't be forgotten

`HandlerOptions.ReplaceAttr` is called for every non-group attribute before it is written, including the built-in `time`/`level`/`msg`/`source` attributes. Returning a zero `Attr` drops the attribute entirely.

```go
var redactKeys = map[string]bool{
    "kubeconfig": true, "token": true, "authorization": true,
    "password": true, "bearer": true, "api_key": true,
}

opts := &slog.HandlerOptions{
    Level: &lvl,
    ReplaceAttr: func(groups []string, a slog.Attr) slog.Attr {
        if redactKeys[strings.ToLower(a.Key)] {
            return slog.String(a.Key, "[REDACTED]")
        }
        // Also normalise the timestamp to whole milliseconds so log lines
        // from different pods sort deterministically in Loki.
        if len(groups) == 0 && a.Key == slog.TimeKey {
            return slog.Time(slog.TimeKey, a.Value.Time().Truncate(time.Millisecond))
        }
        return a
    },
}
```

The reason to do this in the handler rather than at call sites is structural: **a call site you forget is a leak; a handler you configure once is a guarantee.** The moment your exporter takes a `--kubeconfig` path, a bearer token for an upstream billing API, or a cloud credential, every `slog.Error("request failed", "err", err)` is a candidate for leaking that value into a log aggregator that a much wider set of people can read than can read your cluster secrets.

The `groups` parameter is the list of currently open `WithGroup` names, so you can scope a rule to one subsystem. It is never called for the group attribute itself — only for its contents.

### `flag` vs `cobra`

`flag` is the stdlib parser. It is small, has no dependencies, and covers a single-command tool completely.

```go
port := flag.Int("port", 9110, "metrics listen port")
upstream := flag.String("upstream", "", "cost API base URL (required)")
interval := flag.Duration("interval", 30*time.Second, "scrape interval")
level := flag.String("log-level", "info", "debug|info|warn|error")
flag.Parse()

if *upstream == "" {
    fmt.Fprintln(os.Stderr, "error: --upstream is required")
    flag.Usage()   // prints the auto-generated help to stderr
    os.Exit(2)
}
```

Mechanics that differ from `argparse`:

- **`flag.Duration` parses Go duration strings** (`30s`, `2m30s`, `500ms`) via `time.ParseDuration`. You get `--interval=90s` for free; there is no "unit" argument to define.
- **Everything returns a pointer**, because parsing happens later at `flag.Parse()`. The `flag.IntVar(&cfg.Port, ...)` form binds into an existing struct instead, which is usually nicer for a config object.
- **Parsing stops at the first non-flag argument.** `mytool --port=9110 serve --verbose` gives you `port=9110` and leaves `["serve", "--verbose"]` in `flag.Args()` — `--verbose` is *not* parsed. This is the concrete reason subcommands need `cobra` (or manual re-parsing with `flag.NewFlagSet`).
- **`flag` has no required-flag concept**, no short/long pairs (`-p` and `--port` are the same flag, and `--port` and `-port` are both accepted), and no environment-variable binding. Check required values yourself, as above.

Reach for `cobra` (`github.com/spf13/cobra`) when you have verbs — `tool scrape`, `tool serve`, `tool version` — the `kubectl` shape. It gives you a command tree, per-command flags with inheritance, generated help and shell completion, and pairs with `viper` for env/config-file binding. For `gpu-cost-exporter` as specified, `flag` is enough; add `cobra` only if you add a second verb.

### The Prometheus exporter idiom

#### The registry model

`client_golang` has three moving parts, and understanding the split is what makes the rest obvious:

- A **`Collector`** is anything that can produce metrics on demand. It has exactly two methods.
- A **`Registry`** holds a set of Collectors and, on `Gather()`, calls `Collect` on all of them and merges the results, checking for duplicate/inconsistent metric descriptors.
- **`promhttp`** turns a `Gatherer` (the read side of a Registry) into an `http.Handler` that renders the gathered families in the Prometheus text exposition format.

```
  Prometheus server                    your exporter process
  ─────────────────                    ─────────────────────
   GET /metrics ──15s──▶ promhttp.HandlerFor(gatherer, opts)
                              │
                              │ 1. gatherer.Gather()
                              ▼
                        prometheus.Registry
                              │
                              │ 2. for each registered Collector:
                              │       go c.Collect(ch)      ← concurrent
                              ▼
              ┌───────────────┼────────────────┬──────────────────┐
              │               │                │                  │
       GoCollector    ProcessCollector   your GaugeVec      your gpuCollector
       (auto-reg'd)   (auto-reg'd)       (holds values)     (computes NOW)
       go_goroutines  process_cpu_...    gpu_cost_...        ├─ query NVML
       go_memstats_*  process_open_fds   (last Set() value)  ├─ build Desc
                                                             └─ MustNewConstMetric
              │               │                │                  │
              └───────────────┴────────────────┴──────────────────┘
                              │
                              │ 3. merge, dedupe-check, sort
                              ▼
                        text exposition format
                              │
                              ▼
                        HTTP 200, body streamed to Prometheus
```

Note what the diagram shows about `prometheus.DefaultRegisterer`: `client_golang`'s `registry.go` has an `init()` that registers a `ProcessCollector` and a `GoCollector` into the default registry. So `promhttp.Handler()` gives you `go_goroutines`, `go_memstats_*`, `process_cpu_seconds_total`, `process_open_fds`, and `process_resident_memory_bytes` for free — and `go_goroutines` climbing monotonically is precisely how you'd catch the leaked-body/leaked-goroutine bugs from earlier in this lesson, on your own exporter, from your own dashboard.

#### The `promauto` convenience path

```go
var gpuCostEff = promauto.NewGaugeVec(
    prometheus.GaugeOpts{
        Name: "gpu_cost_efficiency_usd_per_gpu_hour",
        Help: "Effective USD cost per GPU-hour by namespace.",
    },
    []string{"namespace"}, // label keys — keep cardinality bounded
)
// in your scrape loop:
gpuCostEff.WithLabelValues("team-a").Set(2.13)
```

`promauto`'s constructors are the `prometheus` package's constructors plus a `MustRegister` into `prometheus.DefaultRegisterer`. They **panic** if registration fails (duplicate name, inconsistent help text) — which is what you want at package-var initialisation time, because a metric registration conflict is a programming error, not a runtime condition. `promauto.With(reg)` returns a `Factory` bound to a specific registry, which is what you use in tests so each test gets a clean registry instead of polluting the global one.

| Type | Semantics | When |
|---|---|---|
| `Counter` | Monotonically increasing; `Inc`/`Add` (non-negative only). Resets to 0 on process restart, which PromQL's `rate()` detects and handles. | Events: scrapes attempted, errors, bytes sent. |
| `Gauge` | Arbitrary up/down; `Set`/`Inc`/`Dec`/`Add`/`Sub`. | Current state: GPUs allocated, $/GPU-hour, queue depth. |
| `Histogram` | Pre-defined buckets; server-side aggregation via `histogram_quantile`. Emits `_bucket`, `_sum`, `_count`. | Latency, sizes — anything where you want quantiles *across* instances. |
| `Summary` | Client-side quantiles. Emits `{quantile="..."}`, `_sum`, `_count`. **Not aggregatable across instances.** | Rarely the right answer in a fleet; prefer `Histogram`. |
| `…Vec` variants | One child metric per unique label-value tuple. `WithLabelValues(...)` returns the child. | Any metric with dimensions. |
| `…Func` variants | A gauge whose value is a callback evaluated at scrape time. | Cheap lazily-computed single values, without a full Collector. |

**Cardinality is arithmetic, not vibes.** The series count for a `…Vec` is the product of the distinct values of every label. Work an example for `gpu-cost-exporter`:

```
  labels: namespace (200 distinct) × node (500 GPU nodes) × gpu_uuid (8 per node)
  series = 200 × 500 × 8 = 800,000 series from ONE metric

  Prometheus in-memory cost, rule of thumb ≈ 3 KiB per active series
    (head chunks + label-set index; varies with label length and churn)
  → 800,000 × 3 KiB ≈ 2.4 GiB of head-block RAM, from one metric.

  Drop gpu_uuid:      200 × 500       = 100,000 series ≈ 300 MiB
  Drop node too:      200             =     200 series ≈ negligible
```

The 3 KiB/series figure is a planning rule of thumb, not a constant — it moves with label-name length, label-value churn, and Prometheus version — but the *shape* is exact: cardinality is multiplicative, so adding one 8-valued label multiplies your bill by 8. Namespace is bounded (it churns slowly, and there are hundreds not millions). A pod name is not: with a training job that restarts, pod names are effectively unbounded over time, and each retired pod's series stays in the head block until it ages out. Never put a pod name, request ID, container ID, timestamp, or error message in a label.

#### The `Desc`-based custom `Collector` path

`promauto.NewGaugeVec` holds state: you `Set` it on a timer, and whatever you last `Set` is what a scrape sees. That means (a) a value can be up to one tick stale, and (b) you're computing metrics whether or not anyone is scraping. Implementing `prometheus.Collector` directly inverts both: values are computed **when the scrape arrives**.

The interface is exactly two methods:

```go
type Collector interface {
    Describe(chan<- *Desc)   // send every Desc this collector can ever produce
    Collect(chan<- Metric)   // send the current metrics; called once per scrape
}
```

The contract, from the interface's own doc comments in `client_golang v1.24.1`:

- `Describe` must send the **same** descriptors every time it is called (idempotent), and must be safe for concurrent use. The registry calls it at registration to detect two collectors claiming the same metric name with different help text or label sets.
- Sending **no** descriptor at all marks the collector "unchecked": registration-time consistency checks are skipped and `Collect` may emit anything. Useful for proxying an upstream whose metric set you don't know ahead of time; dangerous otherwise.
- `Collect` **may be called concurrently** and must be concurrency-safe. Blocking in `Collect` blocks the whole `/metrics` render.
- Metrics sharing a `Desc` must differ in their variable label values — two `MustNewConstMetric` calls with identical labels is a duplicate-metric error at gather time.

```go
type gpuCollector struct {
    src  CostSource // your interface from Lesson 3 — a fake in tests
    desc *prometheus.Desc
}

func newGPUCollector(src CostSource) *gpuCollector {
    return &gpuCollector{
        src: src,
        desc: prometheus.NewDesc(
            "gpu_cost_efficiency_usd_per_gpu_hour",      // fqName
            "Effective USD cost per GPU-hour by namespace.", // help
            []string{"namespace"},                        // variable labels
            prometheus.Labels{"cluster": "us-east-gpu-1"}, // const labels
        ),
    }
}

func (c *gpuCollector) Describe(ch chan<- *prometheus.Desc) { ch <- c.desc }

func (c *gpuCollector) Collect(ch chan<- prometheus.Metric) {
    rows, err := c.src.Costs(context.Background()) // computed NOW, per scrape
    if err != nil {
        // Emit an invalid metric so the registry surfaces the error to the
        // scraper instead of silently exposing a stale or empty result.
        ch <- prometheus.NewInvalidMetric(c.desc, err)
        return
    }
    for _, r := range rows {
        ch <- prometheus.MustNewConstMetric(
            c.desc,                  // *Desc
            prometheus.GaugeValue,   // GaugeValue | CounterValue | UntypedValue
            r.USDPerGPUHour,         // float64
            r.Namespace,             // one arg per variable label, IN ORDER
        )
    }
}
```

The signature to memorise is `MustNewConstMetric(desc *Desc, valueType ValueType, value float64, labelValues ...string) Metric`. Label values are **positional**, matching the order of `variableLabels` in `NewDesc`. Get the order wrong and you get a metric that is syntactically valid and semantically garbage — no compiler help, no runtime error. That's the main cost of this path.

Reach for a custom `Collector` when: the value is expensive and you don't want to compute it unless someone asks (an NVML query, a cloud billing API call); the set of label values is dynamic (namespaces appear and disappear, and a `GaugeVec` would keep emitting the stale child until you `Delete` it); or you're wrapping an upstream that already has its own notion of "current."

Reach for `promauto` + a background tick when: the underlying value is genuinely expensive relative to the scrape interval and you'd rather serve a slightly-stale cached value than make Prometheus wait; or the metric is a counter you increment at the event, not something you can recompute.

#### `promhttp.HandlerFor` and its options

`promhttp.Handler()` is exactly `InstrumentMetricHandler(prometheus.DefaultRegisterer, HandlerFor(prometheus.DefaultGatherer, HandlerOpts{}))` — the default registry, default options, plus `promhttp_metric_handler_requests_total`/`_in_flight` instrumentation on itself. For anything you own, use `HandlerFor` with an explicit registry and explicit options:

| `HandlerOpts` field | Default | Why you'd set it |
|---|---|---|
| `Registry` | nil | Registers `promhttp_metric_handler_errors_total{cause=...}` so gather errors are visible as metrics, not just logs. |
| `ErrorLog` | nil | Without it, collection errors are **silently discarded**. Wire it to your `slog`. |
| `ErrorHandling` | `HTTPErrorOnError` | `ContinueOnError` serves the metrics that did collect and skips the broken ones — usually better than failing the whole scrape for one bad collector. |
| `MaxRequestsInFlight` | 0 (unlimited) | Caps concurrent `/metrics` renders. Two Prometheus replicas plus a curious human plus a debug loop can otherwise run four expensive `Collect`s at once. |
| `Timeout` | 0 (none) | Responds 503 if gathering exceeds it. Note the doc's warning: the underlying `Gather` **keeps running** in the background and its result is thrown away. |
| `CoalesceGather` | false (experimental) | Deduplicates concurrent `Gather` calls so overlapping scrapers share one collection cycle. Pairs with `Timeout`; the docs note a joined request that times out still holds a `MaxRequestsInFlight` slot. |
| `EnableOpenMetrics` | false | Needed for exemplars. Changes `le`/`quantile` label formatting (integers gain a trailing `.0`), which **changes series identity** on the Prometheus server. Don't flip it casually. |
| `DisableCompression` | false | The handler negotiates identity/gzip/zstd by default. |

`ErrorLog` being nil-by-default is the trap: a collector that returns errors on every scrape produces a perfectly quiet exporter with missing metrics.

#### Concurrent collection inside a single scrape

A single exporter usually has to gather several independent things per scrape. dcgm-exporter reads many DCGM fields per GPU; your exporter might hit several upstream cost endpoints. Doing this serially makes scrape latency the **sum** of the parts; doing it concurrently makes it the **max**.

`prometheus/node_exporter` solves this in `collector/collector.go` with a pattern worth copying wholesale. Its internal `Collector` interface has one method, `Update(ch chan<- prometheus.Metric) error`. `NodeCollector.Collect` fans every sub-collector out into its own goroutine under a `sync.WaitGroup` and waits, and an `execute` helper wraps each call to time it and record two self-observability metrics: `node_scrape_collector_duration_seconds` and `node_scrape_collector_success`, both labelled by collector name.

Those two metrics are the part people skip and shouldn't. They mean that when a scrape gets slow you can answer *which sub-collector* got slow with a PromQL query instead of a profiler, and when a metric goes missing you can tell "the collector failed" from "the value is genuinely absent." Build the equivalent into `gpu-cost-exporter` the first time it has more than one source:

```go
// Inside Collect, bounded fan-out — Lesson 4's errgroup, one layer down.
g, ctx := errgroup.WithContext(ctx)
g.SetLimit(8) // never let a burst of sub-collectors outnumber your CPU budget
for name, sub := range c.subs {
    name, sub := name, sub
    g.Go(func() error {
        start := time.Now()
        err := sub.Update(ctx, ch)
        ch <- prometheus.MustNewConstMetric(c.durationDesc,
            prometheus.GaugeValue, time.Since(start).Seconds(), name)
        ch <- prometheus.MustNewConstMetric(c.successDesc,
            prometheus.GaugeValue, boolToFloat(err == nil), name)
        return nil // never fail the whole scrape for one sub-collector
    })
}
_ = g.Wait()
```

Note `return nil`: `errgroup`'s first-error-cancels behaviour is exactly wrong here. One failing sub-collector should degrade one metric, not blank the entire `/metrics` response. That's the `ContinueOnError` philosophy expressed in your own code.

## Perspectives

**Developer view.** The exporter idiom is deliberately unopinionated glue over the stdlib — no framework decision to make, a real productivity win for a single-purpose tool. But there's no framework quietly defaulting a timeout or a body-drain for you: you own all nine clocks, every error path, and every cardinality decision that a batteries-included framework might paper over. The compensating advantage is that there is no magic to debug. When a scrape hangs, the goroutine dump names the exact stdlib function it is parked in, and you can go read that function's source in `$(go env GOROOT)/src`.

**Operator view.** An exporter is infrastructure that other infrastructure depends on. If `gpu-cost-exporter` hangs, it isn't "one dashboard panel is stale" — Prometheus's scrape of that target occupies a scrape slot until its own `scrape_timeout` fires, and a target that reliably eats its full timeout degrades the collection budget for other targets on the same instance. `up{job="gpu-cost-exporter"} == 0` plus a rising `scrape_duration_seconds` is the signature. That's why "always bound the client" is the highest-leverage line in the file, and why exposing `go_goroutines` (free, via the default registry) is the cheapest leak detector you will ever deploy.

**Hardware / data-volume view.** dcgm-exporter — the real tool `gpu-cost-exporter` is modeled after — emits dozens of series per GPU per scrape: utilization, framebuffer memory, SM and Tensor-pipe activity, NVLink and PCIe throughput, power, temperature, ECC counters. Multiply by 8 GPUs per node and 500 nodes and one exporter design decision becomes tens of thousands of series. The exporter's own footprint matters *because it runs as a DaemonSet on every GPU node*: an extra 20 MiB of RSS or 50 ms of CPU per scrape multiplies by node count, and it is competing for the same node resources as the training jobs you're there to measure.

**Economics view.** Label cardinality is a literal line item. Every unique label-value combination is a new time series, costing head-block RAM now and object storage later. The 800,000-series calculation above is not hypothetical arithmetic — it is the difference between a Prometheus that fits in one pod and one that needs sharding, remote-write, and a downsampling story. On managed Prometheus offerings that bill per sample ingested, the same multiplication shows up directly on an invoice. The single highest-leverage cost decision in an exporter is made when you type the `[]string{...}` of label names.

## Real-world use cases

- **Google Cloud — Monitoring GPU workloads on GKE with NVIDIA DCGM.** <https://cloud.google.com/blog/products/containers-kubernetes/monitoring-gpu-workloads-on-gke-with-nvidia-data-center-gpu-manager> — the official walkthrough of the exact real exporter `gpu-cost-exporter` is a toy version of: DCGM Exporter running as a privileged DaemonSet, exporting `DCGM_FI_DEV_GPU_UTIL`, `DCGM_FI_DEV_FB_USED`, SM/Tensor-Core activity, and NVLink/PCIe throughput as Prometheus metrics. **What it shows:** the idiom you just learned, in production, at the exact layer you'll operate — including that the exporter is a DaemonSet, which is what makes per-instance efficiency a fleet-wide number.
- **NVIDIA/dcgm-exporter.** <https://github.com/NVIDIA/dcgm-exporter> — the actual open-source exporter: real Go source, real registration code, a real `Collector` implementation over the DCGM API, and a CSV-driven field config (`-f /etc/dcgm-exporter/*.csv`) that decides which DCGM field IDs become metrics. **What it shows:** how a production exporter externalises the cardinality decision into config rather than hardcoding it — the operational answer to the 800,000-series problem. You'll return to this repo explicitly in Lesson 8.
- **prometheus/node_exporter — `collector/collector.go`.** <https://github.com/prometheus/node_exporter/blob/master/collector/collector.go> — defines a one-method internal `Collector` interface (`Update(ch chan<- prometheus.Metric) error`); `NodeCollector.Collect` fans sub-collectors out into goroutines under a `sync.WaitGroup`, and `execute` records `node_scrape_collector_duration_seconds` and `node_scrape_collector_success` per collector. **What it shows:** the concurrent-collection pattern above, plus the self-observability metrics that make a slow scrape debuggable with PromQL instead of a profiler.
- **Cloudflare — How Cloudflare runs Prometheus at scale.** <https://blog.cloudflare.com/how-cloudflare-runs-prometheus-at-scale/> — an operator's account of running Prometheus across a large fleet, including the memory cost of active series and the failure mode where a single high-cardinality metric destabilises an instance. **What it shows:** the "cardinality is a cost line item" claim, grounded in an organisation that has hit the wall and written down what it looked like.

## Worked example

A complete, runnable `gpu-cost-exporter` core: flags with validation, a runtime-adjustable `slog` JSON logger with redaction, a fully-configured `http.Transport` and `http.Client`, a streaming JSON decode that drains its body, a `Desc`-based custom `Collector`, a hardened `http.Server`, and graceful shutdown on `SIGTERM`.

```go
// Package main implements gpu-cost-exporter: a Prometheus exporter that
// reports effective USD-per-GPU-hour by Kubernetes namespace.
package main

import (
	"context"
	"encoding/json"
	"errors"
	"flag"
	"fmt"
	"io"
	"log/slog"
	"net/http"
	"os"
	"os/signal"
	"strings"
	"syscall"
	"time"

	"github.com/prometheus/client_golang/prometheus"
	"github.com/prometheus/client_golang/prometheus/collectors"
	"github.com/prometheus/client_golang/prometheus/promhttp"
)

// ---------------------------------------------------------------- data source

// nsCost is one row of the upstream cost API's response.
type nsCost struct {
	Namespace     string   `json:"namespace"`
	USDPerGPUHour float64  `json:"usd_per_gpu_hour"`
	// Pointer, not float64: a real 0.00 must be distinguishable from
	// "the upstream did not report this namespace". With a plain float64
	// plus omitempty on the producing side, those two collapse.
	CreditsUSD    *float64 `json:"credits_usd,omitempty"`
}

// CostSource is the consumer-defined interface (Lesson 3). Production uses
// httpCostSource; tests use a hand-written fake, no mocking library.
type CostSource interface {
	Costs(ctx context.Context) ([]nsCost, error)
}

type httpCostSource struct {
	client *http.Client
	url    string
	log    *slog.Logger
}

func (s *httpCostSource) Costs(ctx context.Context) ([]nsCost, error) {
	req, err := http.NewRequestWithContext(ctx, http.MethodGet, s.url, nil)
	if err != nil {
		return nil, fmt.Errorf("build request: %w", err)
	}
	req.Header.Set("Accept", "application/json")

	resp, err := s.client.Do(req)
	if err != nil {
		return nil, fmt.Errorf("scrape %s: %w", s.url, err)
	}
	// Deferred IMMEDIATELY after the error check, before the status check —
	// on a non-2xx, err is nil and Body is a real open stream.
	defer func() {
		// Drain (bounded) so the connection returns to the idle pool; a
		// json.Decoder stops at the end of the first value and leaves the
		// trailing newline unread, which alone would kill reuse.
		_, _ = io.Copy(io.Discard, io.LimitReader(resp.Body, 64<<10))
		_ = resp.Body.Close()
	}()

	if resp.StatusCode != http.StatusOK {
		return nil, fmt.Errorf("scrape %s: unexpected status %d", s.url, resp.StatusCode)
	}

	dec := json.NewDecoder(resp.Body) // streaming: peak RSS is the decoded value
	dec.DisallowUnknownFields()       // an upstream schema change is an error, not silence
	var rows []nsCost
	if err := dec.Decode(&rows); err != nil {
		return nil, fmt.Errorf("decode cost payload: %w", err)
	}
	return rows, nil
}

// ------------------------------------------------------------------ collector

type gpuCollector struct {
	src          CostSource
	timeout      time.Duration
	log          *slog.Logger
	costDesc     *prometheus.Desc
	durationDesc *prometheus.Desc
	successDesc  *prometheus.Desc
}

func newGPUCollector(src CostSource, timeout time.Duration, log *slog.Logger) *gpuCollector {
	return &gpuCollector{
		src:     src,
		timeout: timeout,
		log:     log,
		costDesc: prometheus.NewDesc(
			"gpu_cost_efficiency_usd_per_gpu_hour",
			"Effective USD cost per GPU-hour by namespace.",
			[]string{"namespace"}, nil, // exactly ONE label: bounded cardinality
		),
		durationDesc: prometheus.NewDesc(
			"gpu_cost_scrape_duration_seconds",
			"Duration of the upstream cost API scrape.",
			nil, nil,
		),
		successDesc: prometheus.NewDesc(
			"gpu_cost_scrape_success",
			"1 if the last upstream cost API scrape succeeded, 0 otherwise.",
			nil, nil,
		),
	}
}

// Describe must send the same Descs on every call and be concurrency-safe.
func (c *gpuCollector) Describe(ch chan<- *prometheus.Desc) {
	ch <- c.costDesc
	ch <- c.durationDesc
	ch <- c.successDesc
}

// Collect runs once per scrape and may be called concurrently.
func (c *gpuCollector) Collect(ch chan<- prometheus.Metric) {
	// Bound the work this scrape can do, independent of the client's ceiling.
	ctx, cancel := context.WithTimeout(context.Background(), c.timeout)
	defer cancel()

	start := time.Now()
	rows, err := c.src.Costs(ctx)
	elapsed := time.Since(start)

	// Self-observability first, so it is emitted on both paths.
	ch <- prometheus.MustNewConstMetric(c.durationDesc, prometheus.GaugeValue, elapsed.Seconds())
	if err != nil {
		ch <- prometheus.MustNewConstMetric(c.successDesc, prometheus.GaugeValue, 0)
		c.log.Error("cost scrape failed", "err", err, "duration_ms", elapsed.Milliseconds())
		return // emit NO cost series rather than stale ones
	}
	ch <- prometheus.MustNewConstMetric(c.successDesc, prometheus.GaugeValue, 1)

	for _, r := range rows {
		// Label values are POSITIONAL and match NewDesc's variableLabels order.
		ch <- prometheus.MustNewConstMetric(
			c.costDesc, prometheus.GaugeValue, r.USDPerGPUHour, r.Namespace,
		)
	}
	c.log.Info("cost scrape complete",
		"rows", len(rows), "duration_ms", elapsed.Milliseconds())
}

// ----------------------------------------------------------------------- main

var redactKeys = map[string]bool{
	"kubeconfig": true, "token": true, "authorization": true, "api_key": true,
}

func main() {
	if err := run(); err != nil {
		fmt.Fprintf(os.Stderr, "fatal: %v\n", err)
		os.Exit(1)
	}
}

func run() error {
	port := flag.Int("port", 9110, "metrics listen port")
	upstream := flag.String("upstream", "", "cost API URL (required)")
	scrapeTimeout := flag.Duration("scrape-timeout", 8*time.Second, "per-scrape deadline")
	fanout := flag.Int("fanout", 16, "max concurrent upstream requests")
	logLevel := flag.String("log-level", "info", "debug|info|warn|error")
	flag.Parse()

	if *upstream == "" {
		flag.Usage()
		return errors.New("--upstream is required")
	}

	// Runtime-adjustable level: a *LevelVar is a Leveler, so the handler
	// consults it on every record.
	var lvl slog.LevelVar
	if err := lvl.UnmarshalText([]byte(*logLevel)); err != nil {
		return fmt.Errorf("parse --log-level: %w", err)
	}
	log := slog.New(slog.NewJSONHandler(os.Stdout, &slog.HandlerOptions{
		Level: &lvl,
		// Central redaction: safe by construction at every call site.
		ReplaceAttr: func(_ []string, a slog.Attr) slog.Attr {
			if redactKeys[strings.ToLower(a.Key)] {
				return slog.String(a.Key, "[REDACTED]")
			}
			return a
		},
	})).With("exporter", "gpu-cost")
	slog.SetDefault(log)

	// Transport: the pool lives here, so build ONE and share it.
	tr := http.DefaultTransport.(*http.Transport).Clone()
	tr.MaxIdleConns = 100
	tr.MaxIdleConnsPerHost = *fanout    // default is 2 — too small for fan-out
	tr.IdleConnTimeout = 90 * time.Second
	tr.TLSHandshakeTimeout = 5 * time.Second
	tr.ResponseHeaderTimeout = 5 * time.Second // UNSET by default; the key clock
	tr.ExpectContinueTimeout = 1 * time.Second

	client := &http.Client{
		Transport: tr,
		Timeout:   30 * time.Second, // seatbelt; ctx does the real bounding
	}

	src := &httpCostSource{client: client, url: *upstream, log: log}

	// A private registry, not the global default: explicit, and testable.
	reg := prometheus.NewRegistry()
	reg.MustRegister(
		collectors.NewGoCollector(),      // go_goroutines — your leak detector
		collectors.NewProcessCollector(collectors.ProcessCollectorOpts{}),
		newGPUCollector(src, *scrapeTimeout, log),
	)

	mux := http.NewServeMux()
	mux.Handle("GET /metrics", promhttp.HandlerFor(reg, promhttp.HandlerOpts{
		Registry:            reg,                        // exposes handler error counters
		ErrorLog:            slog.NewLogLogger(log.Handler(), slog.LevelError),
		ErrorHandling:       promhttp.ContinueOnError,   // one bad collector != empty scrape
		MaxRequestsInFlight: 4,
		Timeout:             10 * time.Second,
	}))
	mux.HandleFunc("GET /healthz", func(w http.ResponseWriter, _ *http.Request) {
		w.WriteHeader(http.StatusOK)
		_, _ = io.WriteString(w, "ok\n")
	})
	mux.HandleFunc("PUT /debug/level", func(w http.ResponseWriter, r *http.Request) {
		var l slog.Level
		if err := l.UnmarshalText([]byte(r.URL.Query().Get("v"))); err != nil {
			http.Error(w, err.Error(), http.StatusBadRequest)
			return
		}
		lvl.Set(l)
		log.Info("log level changed", "level", l.String())
	})

	srv := &http.Server{
		Addr:              fmt.Sprintf(":%d", *port),
		Handler:           mux,
		ReadHeaderTimeout: 5 * time.Second,  // slowloris; gosec G112 flags its absence
		ReadTimeout:       10 * time.Second, // headers + body
		WriteTimeout:      15 * time.Second, // > HandlerOpts.Timeout, so 503s can be written
		IdleTimeout:       60 * time.Second, // keep-alive reuse across Prometheus scrapes
		MaxHeaderBytes:    1 << 16,          // 64 KiB; default is 1 MiB
	}

	ctx, stop := signal.NotifyContext(context.Background(), syscall.SIGTERM, syscall.SIGINT)
	defer stop()

	errCh := make(chan error, 1)
	go func() {
		log.Info("listening", "addr", srv.Addr, "upstream", *upstream)
		if err := srv.ListenAndServe(); err != nil && !errors.Is(err, http.ErrServerClosed) {
			errCh <- err
		}
		close(errCh)
	}()

	select {
	case err := <-errCh:
		return err
	case <-ctx.Done():
		log.Info("shutdown signal received, draining")
		shutdownCtx, cancel := context.WithTimeout(context.Background(), 10*time.Second)
		defer cancel()
		if err := srv.Shutdown(shutdownCtx); err != nil {
			return fmt.Errorf("graceful shutdown: %w", err)
		}
		log.Info("shutdown complete")
		return nil
	}
}
```

**Reading the output.** With the upstream returning `[{"namespace":"team-a","usd_per_gpu_hour":2.13},{"namespace":"team-b","usd_per_gpu_hour":0.87}]`:

```
$ curl -s localhost:9110/metrics | grep -E '^(gpu_cost|go_goroutines)'
# HELP gpu_cost_efficiency_usd_per_gpu_hour Effective USD cost per GPU-hour by namespace.
# TYPE gpu_cost_efficiency_usd_per_gpu_hour gauge
gpu_cost_efficiency_usd_per_gpu_hour{namespace="team-a"} 2.13
gpu_cost_efficiency_usd_per_gpu_hour{namespace="team-b"} 0.87
# HELP gpu_cost_scrape_duration_seconds Duration of the upstream cost API scrape.
# TYPE gpu_cost_scrape_duration_seconds gauge
gpu_cost_scrape_duration_seconds 0.0412
# HELP gpu_cost_scrape_success 1 if the last upstream cost API scrape succeeded, 0 otherwise.
# TYPE gpu_cost_scrape_success gauge
gpu_cost_scrape_success 1
# HELP go_goroutines Number of goroutines that currently exist.
# TYPE go_goroutines gauge
go_goroutines 9
```

(Representative transcript — the exact duration and goroutine count vary.) Line by line:

- The `# HELP`/`# TYPE` lines come from the `Desc` you built; the registry renders them once per metric family. If two collectors register the same `fqName` with different help text, `MustRegister` panics at startup — deliberately, so you find it in CI rather than in a confused dashboard.
- Two `gpu_cost_efficiency_usd_per_gpu_hour` series, one per namespace, from one `Desc`. Cardinality = number of namespaces, exactly as designed.
- `gpu_cost_scrape_success 1` is the metric that lets you write `gpu_cost_scrape_success == 0` as an alert. Without it, an upstream failure is indistinguishable from "no namespaces have GPU cost right now," because both render as an absence of `gpu_cost_efficiency_*` series.
- `go_goroutines 9` is the free leak detector. Watch it after wiring in the fan-out: a flat line across hours means your `errgroup` and your body-drains are correct; a monotonic ramp means one of them isn't. `rate(go_goroutines[30m]) > 0` sustained is the alert.

**Verifying the timeout claims yourself.** Point `--upstream` at a black hole and watch the failure become fast and logged rather than silent and permanent:

```
$ nc -l 19999 &                         # accepts the TCP connection, never responds
$ ./gpu-cost-exporter --upstream=http://127.0.0.1:19999/v1 --scrape-timeout=8s &
$ time curl -s localhost:9110/metrics | grep gpu_cost_scrape_success
gpu_cost_scrape_success 0
real    0m5.02s
```

Five seconds, not eight and not forever: `Transport.ResponseHeaderTimeout = 5s` fired first, before the 8 s scrape context and long before the 30 s `Client.Timeout`. Delete that one line, rebuild, and the same command takes 8 s (the ctx deadline). Delete the ctx too and it takes 30 s. Delete `Client.Timeout` as well and it never returns — which is the default state of `http.Get`.

## Practice

**Task.** Turn `gpu-cost-exporter` into a real CLI + exporter.

1. Add a CLI layer with `flag` (or `cobra` if you want a `serve` subcommand): `--port`, `--upstream`, `--interval`, `--log-level`. Validate that `--upstream` is non-empty and exit `2` with usage if not.
2. Configure `log/slog` with a JSON handler whose level comes from a `*slog.LevelVar` bound to `--log-level`. Log every scrape with `namespace`, `gpu_hours`, `usd`, `duration_ms` as structured fields. Add a `ReplaceAttr` that redacts `kubeconfig`/`token`/`authorization`-shaped keys.
3. Register **one** cost metric named `gpu_cost_efficiency_usd_per_gpu_hour` with a single `namespace` label, plus `gpu_cost_scrape_duration_seconds` and `gpu_cost_scrape_success`. Do it once with `promauto` + a ticker, then again as a `Desc`-based custom `Collector`, and write down in a comment which one you shipped and why.
4. Build the `http.Client` on an explicitly-configured `http.Transport` (cloned from `DefaultTransport`), setting at minimum `ResponseHeaderTimeout` and `MaxIdleConnsPerHost`. Give the `Client` a `Timeout`.
5. In the scrape path, `defer` a bounded drain (`io.Copy(io.Discard, io.LimitReader(body, 64<<10))`) **and** `Close`, placed immediately after the `client.Do` error check — before the status-code check.
6. Serve the metrics handler via `promhttp.HandlerFor` with an explicit registry, `ErrorLog`, and `ContinueOnError`, on an explicit `&http.Server{}` with all four timeouts. Add `srv.Shutdown` on `SIGTERM`.

**Acceptance.**

- `curl localhost:PORT/metrics | grep gpu_cost_efficiency` shows the gauge with a `namespace` label and a numeric value, and `gpu_cost_scrape_success 1`.
- Logs are JSON with key/value fields (verify: `... | jq .namespace`), and a log line containing a `token` key renders `[REDACTED]`.
- Blackhole the upstream (`nc -l <port>` with no response). `/metrics` must return within your `ResponseHeaderTimeout`, `gpu_cost_scrape_success` must be `0`, and the process must not accumulate goroutines: `go_goroutines` is flat across ten scrapes against the black hole.
- `go vet` and `golangci-lint` (with `gosec` enabled) surface no missing-timeout finding on the `http.Server` (G112) and no unclosed-body finding.
- `kill -TERM <pid>` logs `shutdown complete` and exits 0 without dropping an in-flight `/metrics` request.

Full task and grading rubric: [`gpu-cost-exporter/README.md`](../practice/gpu-cost-exporter/README.md).

## Common pitfalls

1. **Using `http.Get` / `http.DefaultClient` "just for a quick call."** *Symptom:* the exporter stops scraping and eventually OOMs; `go_goroutines` climbs linearly with time. *Mechanism:* `DefaultClient.Timeout` is the zero value, documented as "no timeout," and `DefaultTransport.ResponseHeaderTimeout` is also unset — so a server that accepts your connection and then goes quiet parks a goroutine and a socket forever. Every tick adds another. Never call `http.Get` in exporter code.

2. **Closing the response body without draining it.** *Symptom:* a latency floor that doesn't improve when the upstream gets faster; a churn of `TIME_WAIT` sockets; handshake CPU on the upstream that tracks your request count. *Mechanism:* `transport.go`'s `readLoop` sets `alive = bodyEOF && !sawEOF && wroteRequest() && tryPutIdleConn(...)`. Closing before EOF makes `bodyEOF` false and the connection is destroyed instead of pooled. `json.Decoder.Decode` stops at the end of the first value and does not reach EOF, so *decoding is not draining*.

3. **Deferring `Body.Close()` after the status check instead of after the error check.** *Symptom:* file-descriptor growth that only appears when the upstream is unhealthy — invisible in staging, visible at 3am. *Mechanism:* on a non-2xx response `err` is `nil` and the body is a real open stream; an early `return` before the `defer` executes leaks it.

4. **Unbounded label cardinality.** *Symptom:* Prometheus head-block RAM growth, then OOM or a refused-to-ingest error; queries over the metric time out. *Mechanism:* series count is the product of distinct values across all labels, so one pod-name label on a fleet-scale metric multiplies your series count by the number of pods that have ever existed in the retention window — pod names are unbounded over time even though they're bounded at any instant.

5. **`omitempty` dropping a real zero.** *Symptom:* a namespace with genuinely zero GPU spend disappears from the payload, and the downstream consumer cannot tell "no data" from "confirmed \$0." *Mechanism:* `omitempty` omits `0`, `""`, `false`, nil, and empty collections — it has no concept of "was set." Use `*float64` when unset and zero mean different things. Note that `omitempty` also does *nothing* on a struct field like `time.Time`; Go 1.24's `omitzero` is the option that handles those.

6. **Not calling `dec.DisallowUnknownFields()` when parsing someone else's API.** *Symptom:* every value reads as zero after an upstream deploy; no error anywhere. *Mechanism:* the decoder skips keys it can't match to a struct field, silently. A vendor renaming `usd_per_gpu_hour` → `usdPerGpuHour` is a clean decode into a zero-valued struct. (And note field matching is case-*insensitive*, so `USD_PER_GPU_HOUR` would still have worked — it's the shape change that kills you.)

7. **Leaving `promhttp.HandlerOpts.ErrorLog` nil.** *Symptom:* metrics are quietly missing from `/metrics` and nothing is logged. *Mechanism:* the handler discards collection errors when `ErrorLog` is nil. Wire it to `slog` via `slog.NewLogLogger(log.Handler(), slog.LevelError)`, and set `Registry` so `promhttp_metric_handler_errors_total` exists as a metric you can alert on.

## Self-check

**(a) How do you expose a Prometheus gauge from Go, and when would you not use `promauto`?**
**Answer:** With `promauto`: construct with `promauto.NewGauge`/`NewGaugeVec` (which calls `MustRegister` on `prometheus.DefaultRegisterer` and panics on conflict), mutate with `.Set` / `.WithLabelValues(...).Set`, and mount `promhttp.Handler()` at `/metrics`. The registry renders every registered collector in text exposition format when scraped. You would *not* use `promauto` when (i) the value is expensive and should only be computed if someone actually scrapes, (ii) the set of label values is dynamic — a `GaugeVec` keeps emitting a child until you explicitly `Delete` it, so a departed namespace lingers forever — or (iii) you want a private registry for testability. In those cases implement `prometheus.Collector` directly: `Describe(ch chan<- *Desc)` sends the fixed descriptors, `Collect(ch chan<- Metric)` runs once per scrape and emits `prometheus.MustNewConstMetric(desc, valueType, value, labelValues...)` with label values positional in `NewDesc`'s order.

**(b) An `slog` structured field vs an `fmt.Sprintf` log line — why does it matter across 40 clusters?**
**Answer:** `slog.Info("scrape done", "namespace", ns, "usd", c)` emits `namespace` and `usd` as first-class JSON keys, so Loki/Elasticsearch can filter and aggregate `namespace="team-a"` across all 40 clusters without regex on free-form text, and the query survives someone rewording the message. `fmt.Sprintf("scrape done namespace=%s", ns)` produces an opaque string you must parse per-format at query time. Structured attributes also enable handler-level policy that free-form strings cannot have: `ReplaceAttr` can redact any attribute named `token` across every call site in the binary, which is impossible once the secret has been formatted into a message string.

**(c) Name every clock that bounds an outbound HTTP request, and say which segment each one covers.**
**Answer:** Six, plus the context. On the `Transport`: `DialContext`'s `net.Dialer.Timeout` (DNS + TCP connect, 30 s in `DefaultTransport`); `TLSHandshakeTimeout` (10 s); `ExpectContinueTimeout` (1 s, only with an `Expect: 100-continue` header); `ResponseHeaderTimeout` (request fully written → response headers received, **unset by default** — this is the one that catches a wedged upstream); `IdleConnTimeout` (90 s, how long a pooled idle connection survives — not a request bound). On the `Client`: `Timeout`, which spans everything including redirects **and the body read**, keeps running after `Do` returns, and is **0 = no timeout** by default. And the `ctx` deadline from `http.NewRequestWithContext`, which covers the same span as `Client.Timeout` but is per-call, composable, and cancellable — so a `SIGTERM` unwinds in-flight requests instead of waiting them out. Nothing on the `Transport` bounds the body read; only `Client.Timeout` or `ctx` does.

**(d) You close every response body and still see connection churn. What's happening?**
**Answer:** Closing is not enough. In `net/http`'s `readLoop`, a connection is returned to the idle pool only when `alive` is true, and `alive` requires `bodyEOF` — i.e. the body was read all the way to `io.EOF`. If you `Close` first, the transport's `earlyCloseFn` fires, `bodyEOF` is false, and the connection is destroyed. The most common way to hit this while believing you consumed the body is `json.NewDecoder(r).Decode(&v)`: it stops at the end of the first JSON value and leaves the trailing newline unread. Fix: `io.Copy(io.Discard, io.LimitReader(resp.Body, 64<<10))` before `Close`. A second, independent cause is `Transport.MaxIdleConnsPerHost`, which defaults to `DefaultMaxIdleConnsPerHost = 2` — with a fan-out of 16 to one host, 14 perfectly good connections are closed by `errTooManyIdleHost` on every cycle.

**(e) `ReadHeaderTimeout` is set but `ReadTimeout` isn't. What's still unbounded server-side, and what is `IdleTimeout` doing?**
**Answer:** The request **body** read is unbounded — `ReadHeaderTimeout` bounds only the request line and headers, and the read deadline is reset once headers are parsed, so a client that sends valid headers then trickles the body holds a goroutine indefinitely. `IdleTimeout` is *also* unbounded here: its documented zero-value behaviour is to fall back to `ReadTimeout`, and if `ReadTimeout` is zero or negative there is no timeout at all — so keep-alive connections accumulate one goroutine and one FD each. The inverse is worth knowing too: setting only `ReadTimeout` gives you `ReadHeaderTimeout` and `IdleTimeout` for free at the same value, which is why "set all four explicitly" is the rule rather than "set the important one."

**(f) Why does node_exporter run its sub-collectors concurrently inside a single `Collect()` call, and what pattern is that?**
**Answer:** Each sub-collector does independent I/O-bound work per scrape, so serial execution makes total scrape latency the *sum* of every sub-collector's latency instead of the *max*. `NodeCollector.Collect` fans them into goroutines under a `sync.WaitGroup`; an `execute` helper times each one and records `node_scrape_collector_duration_seconds` and `node_scrape_collector_success` per collector name, which is what lets you attribute a slow scrape with PromQL. It's the bounded-fan-out pattern from Lesson 4 (`errgroup.SetLimit`) one layer down, *inside* the exporter idiom — with one deliberate difference: you return `nil` from each `g.Go`, because `errgroup`'s first-error-cancels behaviour would blank the whole `/metrics` response for one failing sub-collector.

## Connections & what's next

This lesson is where Lessons 2–6 cash out into running infrastructure: error wrapping (Lesson 2) shows up in every `scrape` return and in classifying a `ctx` deadline as retryable; interfaces-on-the-consumer (Lesson 3) is why `CostSource` is declared next to the collector that uses it and faked without a mocking library in tests; the bounded-fan-out pattern (Lesson 4) reappears *inside* the exporter idiom via concurrent sub-collectors; `context` threading (Lesson 4) is what makes `SIGTERM` unwind an in-flight scrape; table-driven tests over a fake `CostSource` (Lesson 5) are how you test all of it without a network; and module layout (Lesson 6) is the repo shape this code lives in. It also sets up Lesson 9's controller directly — a controller's reconcile loop and an exporter's scrape loop are the same shape (timed, idempotent, must-not-hang), just triggered by a workqueue instead of a ticker.

**Next: Lesson 8, [Reading unfamiliar Go](08-reading-unfamiliar-go.md).** You just built the exporter idiom from scratch; next you go read the *real* source it was modeled on — `go doc`, jump-to-definition, and "read the test first" turn an unfamiliar repo from intimidating to legible in one sitting.

## References & further reading

**Primary sources**

- **`net/http`** — [pkg docs](https://pkg.go.dev/net/http) — the authority for every field in this lesson's `Client`, `Transport`, and `Server` tables, including the zero-value semantics and the `ReadHeaderTimeout`/`IdleTimeout` fallback to `ReadTimeout`. The source is also on your disk: `$(go env GOROOT)/src/net/http/transport.go` contains `DefaultTransport`, `DefaultMaxIdleConnsPerHost = 2`, and the `readLoop`'s `alive = ... && bodyEOF && ...` conjunction that decides connection reuse.
- **`encoding/json`** — [pkg docs](https://pkg.go.dev/encoding/json) — struct-tag grammar, the exact definition of "empty" for `omitempty`, the Go 1.24 `omitzero` option and its `IsZero()` rule, case-insensitive field matching, `float64`-for-`any` number decoding, `UseNumber`, `DisallowUnknownFields`, and `Encoder.SetEscapeHTML`.
- **`log/slog`** — [pkg docs](https://pkg.go.dev/log/slog) — `Handler`, `HandlerOptions` (`Level`, `AddSource`, `ReplaceAttr` including which built-in attributes it sees), `LevelVar`, `With`/`WithGroup`, and the `TimeKey`/`LevelKey`/`MessageKey`/`SourceKey` constants.
- **`flag`** — [pkg docs](https://pkg.go.dev/flag) — parsing semantics, `Duration` string format, `FlagSet` for manual subcommands, and the "parsing stops at the first non-flag argument" rule that pushes you toward `cobra`.
- **prometheus/client_golang** — [repo](https://github.com/prometheus/client_golang) · [pkg docs](https://pkg.go.dev/github.com/prometheus/client_golang/prometheus) — the `Collector` interface contract quoted above, `NewDesc`/`MustNewConstMetric` signatures, `promauto`'s register-and-panic behaviour, and `promhttp.HandlerOpts` (`ErrorLog`, `ErrorHandling`, `MaxRequestsInFlight`, `Timeout`, `CoalesceGather`, `EnableOpenMetrics`). Checked against v1.24.1.
- **Prometheus — metric and label naming, instrumentation best practices** — <https://prometheus.io/docs/practices/naming/> · <https://prometheus.io/docs/practices/instrumentation/> — the `_total`/`_seconds`/`_bytes` suffix conventions this lesson's metric names follow, and the standing guidance on keeping label cardinality bounded.

**Real-world engineering blogs**

- **Google Cloud — Monitoring GPU workloads on GKE with DCGM** — <https://cloud.google.com/blog/products/containers-kubernetes/monitoring-gpu-workloads-on-gke-with-nvidia-data-center-gpu-manager> — what it shows: the exporter idiom in production as a privileged GPU-node DaemonSet, and the specific DCGM field metrics a real GPU fleet collects.
- **NVIDIA/dcgm-exporter** — <https://github.com/NVIDIA/dcgm-exporter> — what it shows: real Go source implementing this lesson's idiom at production scale, with the metric set externalised into a CSV field config rather than hardcoded. The target you'll read critically in Lesson 8.
- **prometheus/node_exporter — `collector/collector.go`** — <https://github.com/prometheus/node_exporter/blob/master/collector/collector.go> — what it shows: the one-method sub-collector interface, the `sync.WaitGroup` fan-out inside `Collect`, and the `node_scrape_collector_duration_seconds` / `node_scrape_collector_success` self-observability metrics. Read once end to end.
- **Cloudflare — How Cloudflare runs Prometheus at scale** — <https://blog.cloudflare.com/how-cloudflare-runs-prometheus-at-scale/> — what it shows: cardinality and fleet-scale scrape economics from an operator that has hit the wall, which is where the per-series memory rule of thumb in this lesson comes from.

**Deeper dives**

- **Go blog — Structured Logging with slog** — <https://go.dev/blog/slog> — the design rationale behind `Handler`/`Attr`/`Record` and why the API is split the way it is; worth a skim once the pkg docs make sense.
- **Go blog — Routing Enhancements for Go 1.22** — <https://go.dev/blog/routing-enhancements> — the `[METHOD ][HOST]/[PATH]` `ServeMux` pattern syntax and precedence rules used in this lesson's worked example, and why they removed most reasons to reach for a third-party router.
- **spf13/cobra** — [repo](https://github.com/spf13/cobra) — command trees, flag inheritance, and generated completions; read only if you add a second verb to the exporter.
