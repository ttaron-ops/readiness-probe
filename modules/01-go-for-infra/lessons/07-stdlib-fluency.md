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
sources: 9
---

# 01.7 · Standard library fluency: net/http, json, slog, flags, and the Prometheus exporter idiom

> **Concept.** The stdlib (`net/http`, `encoding/json`, `log/slog`, `flag`, `time`, `io`, `os`) plus `prometheus/client_golang` is the entire raw material of an exporter or CLI — learn its idioms and gotchas cold.
>
> Module: [🐹 01 — Go for infrastructure engineers](../README.md) · Deliverable: [`gpu-cost-exporter`](../practice/gpu-cost-exporter/README.md)

## Where this fits

Lesson 6 gave you the *container* — `go.mod`, `internal/`, and the layout a controller repo is judged on before anyone reads a line of logic. This lesson gives you what goes *inside* that container: the four stdlib moves (parse flags, log structured, serve HTTP, decode JSON) plus the one third-party idiom you'll use everywhere — `prometheus/client_golang`. By the end you'll have written the actual scrape-and-serve core of `gpu-cost-exporter`, not a toy. That matters immediately, because lesson 8, **Reading unfamiliar Go**, asks you to go read the *real* dcgm-exporter and node_exporter source that this lesson's exporter idiom is modeled on — you can't read those critically until you've built the pattern yourself and felt where it's easy to get wrong.

## Why this matters

Every exporter you operate — node-exporter, kube-state-metrics, dcgm-exporter — is the same four moves: parse flags, log structured JSON, serve HTTP, register metrics. In Python you'd reach for `argparse` + `logging` + `flask` + `prometheus_client` and pull in a web framework. In Go the standard library *is* the framework, and it ships in the binary. A `gpu-cost-exporter` that scrapes GPU allocation and emits `$/GPU-hour` per namespace is ~200 lines of stdlib plus one dependency (`client_golang`). Getting these idioms wrong is how real exporters leak goroutines, hang on a dead upstream, or DoS your own control plane. The single most common production incident in this space is **a client with no timeout** wedging an exporter until it OOMs — you will fix that bug in this lesson so you never ship it. This is also a literal interview signal: CoreWeave- and NVIDIA-class JDs ask for "backend services and Go programs deployed to Kubernetes," and the take-home format for these roles is almost always "write a small tested service" — precisely this shape of code, timeouts and all.

## What's new here (calibration)

Per the module README's skip-list, we do **not** re-teach: what HTTP is, what JSON is, what a CLI flag is, or how to stand up a web framework (Flask/Express-equivalents) — you already know those shapes from Python. We also don't re-derive `argparse`/`logging` basics; you know *why* you'd want flags and structured logs, so we go straight to the Go-specific mechanics and gotchas.

What's genuinely new here, calibrated to staff-level GPU-fleet depth:
- The **exporter idiom** as production infrastructure, not a demo — timeouts, cardinality, and resource footprint as first-class design decisions, because your exporter runs as a DaemonSet on every GPU node in the fleet.
- The difference between `promauto`'s eager, `Set`-on-a-timer convenience wrappers and a **`Desc`-based custom `Collector`** that computes values lazily per-scrape — the pattern real exporters like dcgm-exporter and node_exporter actually use.
- **Concurrent collection inside a single scrape** — the bounded-fan-out pattern from Lesson 4 (`errgroup.SetLimit`) showing up *inside* the exporter idiom itself, exactly as node_exporter implements it.
- Full `http.Server` hardening (four timeouts, not one) and `slog` redaction (`ReplaceAttr`) — the kind of finding a real `gosec`/security review leaves on a PR, not textbook material.

## Core concepts

### From Python to Go: the delta

| You did in Python | You do in Go | The delta that bites |
|---|---|---|
| `requests.get(url)` (default: **no timeout**) | `http.Client{Timeout: 5*time.Second}` | Go's *default* client also has no timeout, but you construct it explicitly, so there's no excuse. `http.Get` uses `DefaultClient` = unbounded. |
| `json.loads` → `dict` | `json.Unmarshal` → struct with tags | No dict-of-anything by default. You declare the shape. Unknown fields are silently dropped, not errored. |
| `logging.info("cost=%s", c)` | `slog.Info("scrape done", "cost", c)` | Key/value pairs, not format strings. Machine-parseable at the source. |
| `argparse` | `flag` (stdlib) or `cobra` (subcommands) | `flag` is tiny and built in; `cobra` for `verb noun` CLIs like `kubectl`. |
| `prometheus_client.Gauge(...).set(x)` | `promauto.NewGaugeVec(...).WithLabelValues(...).Set(x)` | Metrics are registered into a *registry* at construction; label cardinality is your responsibility (and your bill). |
| `flask` route | `http.ServeMux` + `http.HandlerFunc` | No framework. `promhttp.Handler()` is just an `http.Handler` you mount. |

The mental shift: Python gives you dynamic convenience and you bolt on rigor; Go gives you rigor and you can't opt out. Structs-with-tags replace duck typing, explicit timeouts replace hopeful defaults.

### `encoding/json` — tags, `omitempty`, streaming

Struct field tags map Go names to JSON keys. Only exported (capitalized) fields marshal.

```go
type GPUCost struct {
    Namespace string    `json:"namespace"`
    GPUHours  float64   `json:"gpu_hours"`
    USD       float64   `json:"usd,omitempty"` // omit when zero-valued
    Note      string    `json:"-"`             // never marshaled
    Scraped   time.Time `json:"scraped_at"`
}
```

- `omitempty` drops the field when it's the zero value (`0`, `""`, `nil`, `false`) — not when it's "missing." A real `0.0` cost also disappears; that's a gotcha for numeric fields, and it bites hardest when a genuinely idle GPU (real `usd == 0`) silently vanishes from a billing payload instead of reading as "confirmed zero."
- **Streaming**: use `json.NewDecoder(r).Decode(&v)` reading directly from an `io.Reader` (an HTTP body, a file) instead of `Unmarshal(io.ReadAll(...))`. It avoids buffering the whole payload — important scraping a large `/api/v1/pods` response.
- Unknown JSON fields are ignored silently. Call `dec.DisallowUnknownFields()` if you want strictness — fine to skip when parsing your *own* exporter's output, but a real risk when parsing an upstream billing API's response: a field you expected to be authoritative can be silently absent from what you decoded, and you won't know unless you asked to be told.

### `net/http` — server, fully hardened

A handler is anything implementing `ServeHTTP`. `http.HandlerFunc` adapts a plain function.

```go
mux := http.NewServeMux()
mux.Handle("/metrics", promhttp.Handler())
mux.HandleFunc("/healthz", func(w http.ResponseWriter, r *http.Request) {
    w.WriteHeader(http.StatusOK)
    _, _ = io.WriteString(w, "ok\n")
})
srv := &http.Server{
    Addr:              ":9110",
    Handler:           mux,
    ReadHeaderTimeout: 5 * time.Second,  // defends against slowloris; gosec flags its absence
    ReadTimeout:       10 * time.Second, // full request (headers + body) must land in this window
    WriteTimeout:      10 * time.Second, // response must be written in this window
    IdleTimeout:       60 * time.Second, // keep-alive connections don't sit open forever
}
log.Fatal(srv.ListenAndServe())
```

Prefer an explicit `&http.Server{}` over `http.ListenAndServe(addr, mux)` precisely so you can set these. `ReadHeaderTimeout` alone (the field most tutorials mention) only bounds the *header* read — a client that trickles the body, or one your handler is slow to write a response to, is still unbounded without `ReadTimeout`/`WriteTimeout`/`IdleTimeout`. Real PRs get a `gosec`/lint finding for exactly this: set all four unless you have a specific, commented reason not to.

### `net/http` — client. Set a timeout, always.

```go
client := &http.Client{Timeout: 5 * time.Second}
req, _ := http.NewRequestWithContext(ctx, http.MethodGet, url, nil)
resp, err := client.Do(req)
if err != nil { return fmt.Errorf("scrape %s: %w", url, err) }
defer resp.Body.Close() // ALWAYS close, or you leak the connection
```

`Client.Timeout` is a whole-request deadline (dial + redirects + reading the body). A per-attempt bound comes from `ctx` via `context.WithTimeout`. `resp.Body.Close()` is mandatory even on error-adjacent paths where `err == nil` but the status code is bad — an unclosed body pins the TCP connection and defeats keep-alive pooling. It's a slow leak, not a crash, which is exactly why it survives code review: nothing breaks until connection pool exhaustion shows up as an unrelated-looking latency spike days later.

### `log/slog` — structured logging, including redaction

```go
logger := slog.New(slog.NewJSONHandler(os.Stdout, &slog.HandlerOptions{Level: slog.LevelInfo}))
slog.SetDefault(logger)
slog.Info("scrape complete",
    "namespace", ns, "gpu_hours", h, "usd", cost, "duration_ms", took.Milliseconds())
// {"time":"...","level":"INFO","msg":"scrape complete","namespace":"team-a","gpu_hours":12.5,...}
```

Attach persistent context with `logger.With("cluster", name)`. Key/value pairs (not `fmt.Sprintf`) mean Loki/Elasticsearch can filter `namespace="team-a"` without regex.

`slog.HandlerOptions` has a second field worth knowing beyond `Level`: **`ReplaceAttr`**, a function called for every attribute before it's written, letting you rename or scrub fields at the handler level:

```go
opts := &slog.HandlerOptions{
    Level: slog.LevelInfo,
    ReplaceAttr: func(groups []string, a slog.Attr) slog.Attr {
        if a.Key == "kubeconfig" {
            return slog.String("kubeconfig", "[redacted]")
        }
        return a
    },
}
```

This is a real production concern the moment your exporter touches anything credential-shaped — a `--kubeconfig` path, a bearer token, an upstream API key logged on error. Doing redaction centrally in the handler means every call site is safe by construction, instead of hoping every `slog.Info` call remembers to scrub.

### `flag` vs `cobra`

For a single-command tool, `flag` is enough:

```go
port := flag.Int("port", 9110, "metrics listen port")
kubeconfig := flag.String("kubeconfig", "", "path to kubeconfig")
flag.Parse()
```

`cobra` (`github.com/spf13/cobra`) is for `tool scrape`, `tool serve`, `tool version` — the `kubectl` shape. It gives you subcommands, flag inheritance, and generated help. Reach for it when you have verbs.

### The Prometheus exporter idiom

**The `promauto` convenience path.** Construct metrics with `promauto` (auto-registers into the default registry), set values, mount `promhttp.Handler()`.

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

`GaugeVec` = one gauge per label-value combination. `.WithLabelValues("team-a")` returns the child gauge; `.Set` / `.Inc` / `.Add` mutate it. **Cardinality discipline**: never put pod names, request IDs, or timestamps in labels — each unique combination is a new time series and a line item on your Prometheus bill. Namespace is bounded; a pod name is not.

**The `Desc`-based custom `Collector` path.** `promauto.NewGaugeVec` eagerly holds state you `Set` on a timer. The alternative — implementing `prometheus.Collector` directly — is closer to how dcgm-exporter and node_exporter actually work, because it computes values **lazily, per scrape**, rather than maintaining a background ticker:

```go
type gpuCollector struct {
    desc *prometheus.Desc
}

func newGPUCollector() *gpuCollector {
    return &gpuCollector{
        desc: prometheus.NewDesc(
            "gpu_cost_efficiency_usd_per_gpu_hour",
            "Effective USD cost per GPU-hour by namespace.",
            []string{"namespace"}, nil,
        ),
    }
}

func (c *gpuCollector) Describe(ch chan<- *prometheus.Desc) { ch <- c.desc }

func (c *gpuCollector) Collect(ch chan<- prometheus.Metric) {
    for _, ns := range liveNamespaceCosts() { // computed NOW, not on a background timer
        ch <- prometheus.MustNewConstMetric(c.desc, prometheus.GaugeValue, ns.USDPerGPUHour, ns.Namespace)
    }
}
```

`Describe` declares the metric shape up front (Prometheus uses it to detect duplicate/inconsistent registrations); `Collect` is called *once per scrape* and does the real work then. Reach for this when a metric is expensive to compute or depends on live state you don't want to poll on a separate ticker — e.g. querying NVML directly inside `Collect` rather than caching a value that might be stale by the time it's scraped.

### Concurrent collection inside a single exporter

A single exporter often has to gather several independent things per scrape — DCGM-exporter reads dozens of NVML fields per GPU; your own exporter might hit several upstream cost endpoints. `prometheus/node_exporter`'s `NodeCollector` runs each sub-collector's `Update()` in its own goroutine *inside a single `Collect()` call*, then aggregates results (and tracks each sub-collector's own duration/success as metrics in its own right) — see `collector.go` in the References below. This is the exact bounded-fan-out pattern from Lesson 4 (`errgroup.SetLimit`), just showing up one layer down, *inside* the exporter idiom rather than around it. If `gpu-cost-exporter` ever needs to scrape multiple upstreams per cycle, this is the shape: fan out with a bounded `errgroup`, collect results, then `Set`/emit — never let one slow upstream serialize the whole scrape.

## Perspectives

**Developer view.** The exporter idiom (`promauto` + `promhttp.Handler()` + a `GaugeVec`) is deliberately unopinionated glue over the stdlib — no framework decision to make, a real productivity win for a single-purpose tool. But that also means there's no framework quietly defaulting a timeout or a body-close for you: you own every timeout, every error path, and every cardinality decision that a batteries-included framework might otherwise paper over.

**Operator view.** An exporter is infrastructure that other infrastructure depends on. If `gpu-cost-exporter` hangs, it's not just "one dashboard panel is stale" — a wedged scrape target can cascade into Prometheus's own scrape budget being consumed retrying it, degrading collection for *other* targets on the same Prometheus instance. That's why "always set a client timeout" is the single highest-leverage line of code in the whole file.

**Hardware / data-volume view.** DCGM-exporter — the real tool `gpu-cost-exporter` is modeled after — emits dozens of metrics per GPU per scrape: utilization, framebuffer memory used, SM/Tensor Core activity, NVLink and PCIe throughput. At fleet scale, the exporter's own resource footprint matters *because it runs as a DaemonSet on every GPU node* — an inefficiency in one exporter binary multiplies by node count, not by "one server somewhere."

**Economics view.** Label cardinality is a literal line item. Every unique label-value combination is a new Prometheus time series; an accidental high-cardinality label (a pod name, a request ID) on a GPU-fleet-scale exporter is a Prometheus TSDB — or managed-Prometheus billing — cost incident, not just a style nit caught in code review.

## Real-world use cases

- **Google Cloud — Monitoring GPU workloads on GKE with NVIDIA DCGM.** <https://cloud.google.com/blog/products/containers-kubernetes/monitoring-gpu-workloads-on-gke-with-nvidia-data-center-gpu-manager> — official walkthrough of the exact real exporter `gpu-cost-exporter` is a toy version of: DCGM Exporter running as a privileged DaemonSet, exporting `DCGM_FI_DEV_GPU_UTIL`, `DCGM_FI_DEV_FB_USED`, SM/Tensor Core activity, and NVLink/PCIe throughput as Prometheus metrics. Shows the idiom you just learned, in production, at the exact layer you'll operate.
- **NVIDIA/dcgm-exporter (official repo).** <https://github.com/NVIDIA/dcgm-exporter> — the actual open-source exporter: real Go source, real Prometheus registration code. A genuine "read the source of the tool you operate" target — you'll return to this repo explicitly in Lesson 8.
- **prometheus/node_exporter — Collector interface design.** <https://github.com/prometheus/node_exporter/blob/master/collector/collector.go> — the `Collector` interface requires one method (`Update(ch chan<- prometheus.Metric) error`); `NodeCollector` orchestrates many sub-collectors concurrently via goroutines, tracking per-collector duration/success as its own metrics. A second, independent real-world reference implementation of the exporter idiom — and the concrete source for the "concurrent collection inside a scrape" pattern above.
- **Cloudflare — How Cloudflare runs Prometheus at scale.** <https://blog.cloudflare.com/how-cloudflare-runs-prometheus-at-scale/> — a widely-cited operator's account of running Prometheus at real fleet scale; grounds the "cardinality is a cost line item" claim in an organization that has actually hit that wall.

## Worked example

A minimal but complete `gpu-cost-exporter` core: flags, slog, a timed HTTP client scraping an upstream cost API, a `GaugeVec`, and `/metrics` served by a fully-hardened `http.Server`.

```go
package main

import (
    "context"
    "encoding/json"
    "flag"
    "fmt"
    "log/slog"
    "net/http"
    "os"
    "time"

    "github.com/prometheus/client_golang/prometheus"
    "github.com/prometheus/client_golang/prometheus/promauto"
    "github.com/prometheus/client_golang/prometheus/promhttp"
)

var gpuCostEff = promauto.NewGaugeVec(
    prometheus.GaugeOpts{
        Name: "gpu_cost_efficiency_usd_per_gpu_hour",
        Help: "Effective USD cost per GPU-hour by namespace.",
    },
    []string{"namespace"},
)

type nsCost struct {
    Namespace     string  `json:"namespace"`
    USDPerGPUHour float64 `json:"usd_per_gpu_hour"`
}

func scrape(ctx context.Context, client *http.Client, url string, log *slog.Logger) error {
    req, err := http.NewRequestWithContext(ctx, http.MethodGet, url, nil)
    if err != nil {
        return fmt.Errorf("build request: %w", err)
    }
    start := time.Now()
    resp, err := client.Do(req)
    if err != nil {
        return fmt.Errorf("scrape %s: %w", url, err)
    }
    defer resp.Body.Close()
    if resp.StatusCode != http.StatusOK {
        return fmt.Errorf("scrape %s: status %d", url, resp.StatusCode)
    }

    var rows []nsCost
    if err := json.NewDecoder(resp.Body).Decode(&rows); err != nil { // streaming decode
        return fmt.Errorf("decode: %w", err)
    }
    for _, r := range rows {
        gpuCostEff.WithLabelValues(r.Namespace).Set(r.USDPerGPUHour)
    }
    log.Info("scrape complete", "rows", len(rows), "duration_ms", time.Since(start).Milliseconds())
    return nil
}

func main() {
    port := flag.Int("port", 9110, "metrics listen port")
    upstream := flag.String("upstream", "http://cost-api/v1/gpu-cost", "cost API URL")
    interval := flag.Duration("interval", 30*time.Second, "scrape interval")
    flag.Parse()

    log := slog.New(slog.NewJSONHandler(os.Stdout, &slog.HandlerOptions{Level: slog.LevelInfo}))
    slog.SetDefault(log)

    client := &http.Client{Timeout: 5 * time.Second} // MUST: never unbounded

    go func() {
        t := time.NewTicker(*interval)
        defer t.Stop()
        for {
            ctx, cancel := context.WithTimeout(context.Background(), 4*time.Second)
            if err := scrape(ctx, client, *upstream, log); err != nil {
                log.Error("scrape failed", "err", err)
            }
            cancel()
            <-t.C
        }
    }()

    mux := http.NewServeMux()
    mux.Handle("/metrics", promhttp.Handler())
    srv := &http.Server{
        Addr:              fmt.Sprintf(":%d", *port),
        Handler:           mux,
        ReadHeaderTimeout: 5 * time.Second,
        ReadTimeout:       10 * time.Second,
        WriteTimeout:      10 * time.Second,
        IdleTimeout:       60 * time.Second,
    }
    log.Info("listening", "addr", srv.Addr)
    log.Error("server exited", "err", srv.ListenAndServe())
}
```

`curl localhost:9110/metrics` prints `gpu_cost_efficiency_usd_per_gpu_hour{namespace="team-a"} 2.13`. Note the two nested *client-side* timeouts — `Client.Timeout` (5s hard ceiling) and per-scrape `ctx` (4s), which also propagates cancellation if you later wire in shutdown signals — plus the four *server-side* timeouts on `http.Server`, so neither direction of traffic can hang the process.

## Practice

**Task.** Turn `gpu-cost-exporter` into a real CLI + exporter.

1. Add a CLI layer with `flag` (or `cobra` if you want a `serve` subcommand): `--port`, `--upstream`, `--interval`, `--log-level`.
2. Configure `log/slog` with a JSON handler at the chosen level; log every scrape with `namespace`, `gpu_hours`, `usd`, `duration_ms` as structured fields. Add a `ReplaceAttr` that redacts any `kubeconfig`/token-shaped field if you log one.
3. Register **one** `GaugeVec` named `gpu_cost_efficiency_usd_per_gpu_hour` with a single `namespace` label, and `Set` it from scraped data. (Optional stretch: reimplement it as a `Desc`-based custom `Collector` and compare.)
4. Serve `promhttp.Handler()` on `/metrics` via an explicit `&http.Server{}` with all four timeouts: `ReadHeaderTimeout`, `ReadTimeout`, `WriteTimeout`, `IdleTimeout`.
5. Give the scrape's `http.Client` a `Timeout`.

**Acceptance.**
- `curl localhost:PORT/metrics | grep gpu_cost_efficiency` shows the gauge with a `namespace` label and a numeric value.
- Logs are JSON with key/value fields (verify: `... | jq .namespace`).
- The scrape client has a non-zero `Timeout` (point to the line); killing/blackholing the upstream must not hang the exporter past the timeout.
- `go vet` / a lint pass surfaces no missing-timeout finding on the `http.Server`.

Full task and grading rubric: [`gpu-cost-exporter/README.md`](../practice/gpu-cost-exporter/README.md).

## Common pitfalls

1. **Using `http.Get`/`http.DefaultClient` "just for a quick call."** Go's default client has no timeout — the identical failure mode as Python's `requests.get` default — but Go gives you no excuse, since you always construct the client explicitly. Never call `http.Get` in exporter code.
2. **Unbounded label cardinality.** Putting a pod name, a request ID, or a raw timestamp in a label turns a bounded metric into an unbounded one. At GPU-fleet scale this is a cost incident, not a style issue.
3. **Forgetting `resp.Body.Close()` on every path, including error-adjacent ones** (e.g. a non-200 status where `err == nil`). An unclosed body pins the TCP connection and defeats keep-alive pooling — a slow leak that shows up as latency, not a crash.
4. **`omitempty` dropping a real zero-valued cost field.** A genuinely idle GPU with `usd: 0` silently disappears from JSON output — bites hardest at the JSON layer when a downstream billing consumer expects to see the confirmed zero (cross-reference Lesson 1's zero-value semantics).
5. **Not calling `dec.DisallowUnknownFields()` when strictness matters.** Silently ignoring unexpected JSON fields is fine for your own exporter's output but dangerous when parsing an upstream billing API's response — a schema change upstream can silently drop a field you were relying on.

## Self-check

**(a) How do you expose a Prometheus gauge from Go?**
**Answer:** Construct it with `promauto.NewGauge`/`NewGaugeVec` (which registers it into the default registry), mutate it with `.Set`/`.WithLabelValues(...).Set`, and mount `promhttp.Handler()` (an `http.Handler`) at `/metrics` on your `ServeMux`. Scrapers hit that endpoint and the registry renders every registered metric in text format. For lazily-computed values, implement `prometheus.Collector` directly (`Describe`/`Collect`) instead of the `promauto` convenience wrapper.

**(b) An slog structured field vs an `fmt.Sprintf` log line — why does it matter across 40 clusters?**
**Answer:** `slog.Info("scrape done", "namespace", ns, "usd", c)` emits `namespace` and `usd` as first-class JSON keys, so Loki/Elasticsearch can filter and aggregate `namespace="team-a"` across all 40 clusters without brittle regex on free-form strings. `fmt.Sprintf("scrape done namespace=%s", ns)` produces an opaque message you'd have to parse per-format at query time — it breaks the moment someone tweaks the wording, and you can't reliably `sum by (namespace)` over your logs.

**(c) How do you set an `http.Client` timeout and why must you?**
**Answer:** `client := &http.Client{Timeout: 5 * time.Second}` (optionally tighter per-request via `context.WithTimeout` + `http.NewRequestWithContext`). You must because Go's default client — and `http.Get` — has **no** timeout: a slow or hung upstream (a wedged cost API, a dead node) leaves the request blocked forever, goroutines and connections accumulate, and the exporter eventually OOMs or stops scraping. A bounded timeout turns a hang into a fast, logged error.

**(d) Why does `node_exporter` run its sub-collectors concurrently inside a single `Collect()` call, and what pattern is that?**
**Answer:** Each sub-collector (CPU, disk, network, …) does independent I/O-bound work per scrape; running them sequentially would make total scrape latency the *sum* of every sub-collector's latency instead of the *max*. `NodeCollector` fans them out into goroutines and aggregates, which is exactly the bounded-fan-out pattern from Lesson 4 (`errgroup.SetLimit`) — the same concurrency discipline you use to fan out N HTTP calls shows up one layer down, inside the exporter idiom itself.

## Connections & what's next

This lesson is where Lessons 2–6 cash out into actual running infrastructure: error wrapping (Lesson 2) shows up in every `scrape` return; the bounded-fan-out pattern (Lesson 4) reappears *inside* the exporter idiom via `node_exporter`'s concurrent collectors; module layout (Lesson 6) is the repo shape this code lives in. It also sets up Lesson 9's controller directly — a controller's reconcile loop and an exporter's scrape loop are the same shape (timed, idempotent, must-not-hang), just triggered by a workqueue instead of a ticker.

**Next: Lesson 8, [Reading unfamiliar Go](08-reading-unfamiliar-go.md).** You just built the exporter idiom from scratch; next you go read the *real* dcgm-exporter and node_exporter source this lesson modeled it on — `go doc`, jump-to-definition, and "read the test first" turn that source from intimidating to legible in one sitting.

## References & further reading

**Primary sources**
- **prometheus/client_golang** — [repo](https://github.com/prometheus/client_golang) · [pkg docs](https://pkg.go.dev/github.com/prometheus/client_golang/prometheus) — read for the exact `promauto`/`promhttp`/`GaugeVec`/`Collector`/`Desc` API used throughout this lesson.
- **`log/slog`** — [pkg docs](https://pkg.go.dev/log/slog) — read for `Handler`, `HandlerOptions`, `ReplaceAttr`, and `With`.
- **`net/http` & `encoding/json`** — [net/http](https://pkg.go.dev/net/http) · [encoding/json](https://pkg.go.dev/encoding/json) — read for `Server`/`Client` fields (all four timeouts) and struct-tag/streaming/`DisallowUnknownFields` semantics.
- **spf13/cobra** — [repo](https://github.com/spf13/cobra) — read for command trees and flag binding, only if you go the subcommand route.

**Real-world engineering blogs**
- **Google Cloud — Monitoring GPU workloads on GKE with DCGM** — <https://cloud.google.com/blog/products/containers-kubernetes/monitoring-gpu-workloads-on-gke-with-nvidia-data-center-gpu-manager> — what it shows: the real exporter idiom, in production, as a privileged GPU-node DaemonSet.
- **NVIDIA/dcgm-exporter** — <https://github.com/NVIDIA/dcgm-exporter> — what it shows: real Go source implementing this lesson's idiom at production scale; the target you'll read critically in Lesson 8.
- **Cloudflare — How Cloudflare runs Prometheus at scale** — <https://blog.cloudflare.com/how-cloudflare-runs-prometheus-at-scale/> — what it shows: cardinality and fleet-scale scrape economics from an operator that has actually hit the wall.

**Deeper dives**
- **Go blog — structured logging with `slog`** — <https://go.dev/blog/slog> — the design rationale behind `Handler`/`Attr`, worth a skim once the pkg docs make sense.
- **prometheus/node_exporter — `collector.go`** — <https://github.com/prometheus/node_exporter/blob/master/collector/collector.go> — the source for the `Collector` interface and the concurrent-sub-collector pattern discussed above; read once end to end.
