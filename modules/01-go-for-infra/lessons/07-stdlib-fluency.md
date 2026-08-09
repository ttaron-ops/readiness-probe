---
lesson: "01.7"
title: "Standard library fluency: net/http, json, slog, flags, and the Prometheus exporter idiom"
module: "01"
concept: "Standard library fluency: net/http, json, slog, flags, and the Prometheus exporter idiom"
status: not-started
est_time: "10h"
artifacts: []
---

# 01.7 · Standard library fluency: net/http, json, slog, flags, and the Prometheus exporter idiom

> **Concept.** The stdlib (`net/http`, `encoding/json`, `log/slog`, `flag`, `time`, `io`, `os`) plus `prometheus/client_golang` is the entire raw material of an exporter or CLI — learn its idioms and gotchas cold.
>
> Module: [🐹 01 — Go for infrastructure engineers](../README.md) · Deliverable: [`gpu-cost-exporter`](../practice/gpu-cost-exporter/README.md)

## Why this matters

Every exporter you operate — node-exporter, kube-state-metrics, dcgm-exporter — is the same four moves: parse flags, log structured JSON, serve HTTP, register metrics. In Python you'd reach for `argparse` + `logging` + `flask` + `prometheus_client` and pull in a web framework. In Go the standard library *is* the framework, and it ships in the binary. A `gpu-cost-exporter` that scrapes GPU allocation and emits `$/GPU-hour` per namespace is ~200 lines of stdlib plus one dependency (`client_golang`). Getting these idioms wrong is how real exporters leak goroutines, hang on a dead upstream, or DoS your own control plane. The single most common production incident in this space is **a client with no timeout** wedging an exporter until it OOMs — you will fix that bug in this lesson so you never ship it.

## From Python to Go

| You did in Python | You do in Go | The delta that bites |
|---|---|---|
| `requests.get(url)` (default: **no timeout**) | `http.Client{Timeout: 5*time.Second}` | Go's *default* client also has no timeout, but you construct it explicitly, so there's no excuse. `http.Get` uses `DefaultClient` = unbounded. |
| `json.loads` → `dict` | `json.Unmarshal` → struct with tags | No dict-of-anything by default. You declare the shape. Unknown fields are silently dropped, not errored. |
| `logging.info("cost=%s", c)` | `slog.Info("scrape done", "cost", c)` | Key/value pairs, not format strings. Machine-parseable at the source. |
| `argparse` | `flag` (stdlib) or `cobra` (subcommands) | `flag` is tiny and built in; `cobra` for `verb noun` CLIs like `kubectl`. |
| `prometheus_client.Gauge(...).set(x)` | `promauto.NewGaugeVec(...).WithLabelValues(...).Set(x)` | Metrics are registered into a *registry* at construction; label cardinality is your responsibility (and your bill). |
| `flask` route | `http.ServeMux` + `http.HandlerFunc` | No framework. `promhttp.Handler()` is just an `http.Handler` you mount. |

The mental shift: Python gives you dynamic convenience and you bolt on rigor; Go gives you rigor and you can't opt out. Structs-with-tags replace duck typing, explicit timeouts replace hopeful defaults.

## Core notes

**`encoding/json` — tags, `omitempty`, streaming.** Struct field tags map Go names to JSON keys. Only exported (capitalized) fields marshal.

```go
type GPUCost struct {
    Namespace string    `json:"namespace"`
    GPUHours  float64   `json:"gpu_hours"`
    USD       float64   `json:"usd,omitempty"` // omit when zero-valued
    Note      string    `json:"-"`             // never marshaled
    Scraped   time.Time `json:"scraped_at"`
}
```

- `omitempty` drops the field when it's the zero value (`0`, `""`, `nil`, `false`) — not when it's "missing." A real `0.0` cost also disappears; that's a gotcha for numeric fields.
- **Streaming**: use `json.NewDecoder(r).Decode(&v)` reading directly from an `io.Reader` (an HTTP body, a file) instead of `Unmarshal(io.ReadAll(...))`. It avoids buffering the whole payload — important scraping a large `/api/v1/pods` response.
- Unknown JSON fields are ignored silently. Call `dec.DisallowUnknownFields()` if you want strictness.

**`net/http` — server.** A handler is anything implementing `ServeHTTP`. `http.HandlerFunc` adapts a plain function.

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
    ReadHeaderTimeout: 5 * time.Second, // defends against slowloris; gosec flags its absence
}
log.Fatal(srv.ListenAndServe())
```

Prefer an explicit `&http.Server{}` over `http.ListenAndServe(addr, mux)` precisely so you can set `ReadHeaderTimeout`.

**`net/http` — client. Set a timeout, always.**

```go
client := &http.Client{Timeout: 5 * time.Second}
req, _ := http.NewRequestWithContext(ctx, http.MethodGet, url, nil)
resp, err := client.Do(req)
if err != nil { return fmt.Errorf("scrape %s: %w", url, err) }
defer resp.Body.Close() // ALWAYS close, or you leak the connection
```

`Client.Timeout` is a whole-request deadline (dial + redirects + reading the body). A per-attempt bound comes from `ctx` via `context.WithTimeout`. `resp.Body.Close()` is mandatory even on error paths where `err == nil` — an unclosed body pins the TCP connection and defeats keep-alive pooling.

**`log/slog` — structured logging.**

```go
logger := slog.New(slog.NewJSONHandler(os.Stdout, &slog.HandlerOptions{Level: slog.LevelInfo}))
slog.SetDefault(logger)
slog.Info("scrape complete",
    "namespace", ns, "gpu_hours", h, "usd", cost, "duration_ms", took.Milliseconds())
// {"time":"...","level":"INFO","msg":"scrape complete","namespace":"team-a","gpu_hours":12.5,...}
```

Attach persistent context with `logger.With("cluster", name)`. Key/value pairs (not `fmt.Sprintf`) mean Loki/Elasticsearch can filter `namespace="team-a"` without regex.

**`flag` vs `cobra`.** For a single-command tool, `flag` is enough:

```go
port := flag.Int("port", 9110, "metrics listen port")
kubeconfig := flag.String("kubeconfig", "", "path to kubeconfig")
flag.Parse()
```

`cobra` (`github.com/spf13/cobra`) is for `tool scrape`, `tool serve`, `tool version` — the `kubectl` shape. It gives you subcommands, flag inheritance, and generated help. Reach for it when you have verbs.

**The Prometheus exporter idiom.** Construct metrics with `promauto` (auto-registers into the default registry), set values, mount `promhttp.Handler()`.

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

## Worked example

A minimal but complete `gpu-cost-exporter` core: flags, slog, a timed HTTP client scraping an upstream cost API, a `GaugeVec`, and `/metrics`.

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
    Namespace string  `json:"namespace"`
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
    srv := &http.Server{Addr: fmt.Sprintf(":%d", *port), Handler: mux, ReadHeaderTimeout: 5 * time.Second}
    log.Info("listening", "addr", srv.Addr)
    log.Error("server exited", "err", srv.ListenAndServe())
}
```

`curl localhost:9110/metrics` prints `gpu_cost_efficiency_usd_per_gpu_hour{namespace="team-a"} 2.13`. Note the two nested timeouts: `Client.Timeout` (5s hard ceiling) and per-scrape `ctx` (4s) — the context also propagates cancellation if you later wire in shutdown signals.

## Practice

**Task.** Turn `gpu-cost-exporter` into a real CLI + exporter.

1. Add a CLI layer with `flag` (or `cobra` if you want a `serve` subcommand): `--port`, `--upstream`, `--interval`, `--log-level`.
2. Configure `log/slog` with a JSON handler at the chosen level; log every scrape with `namespace`, `gpu_hours`, `usd`, `duration_ms` as structured fields.
3. Register **one** `GaugeVec` named `gpu_cost_efficiency_usd_per_gpu_hour` with a single `namespace` label, and `Set` it from scraped data.
4. Serve `promhttp.Handler()` on `/metrics` via an explicit `&http.Server{}` with `ReadHeaderTimeout`.
5. Give the scrape's `http.Client` a `Timeout`.

**Acceptance.**
- `curl localhost:PORT/metrics | grep gpu_cost_efficiency` shows the gauge with a `namespace` label and a numeric value.
- Logs are JSON with key/value fields (verify: `... | jq .namespace`).
- The scrape client has a non-zero `Timeout` (point to the line); killing/blackholing the upstream must not hang the exporter past the timeout.

## Self-check

**(a) How do you expose a Prometheus gauge from Go?**
**Answer:** Construct it with `promauto.NewGauge`/`NewGaugeVec` (which registers it into the default registry), mutate it with `.Set`/`.WithLabelValues(...).Set`, and mount `promhttp.Handler()` (an `http.Handler`) at `/metrics` on your `ServeMux`. Scrapers hit that endpoint and the registry renders every registered metric in text format.

**(b) An slog structured field vs an `fmt.Sprintf` log line — why does it matter across 40 clusters?**
**Answer:** `slog.Info("scrape done", "namespace", ns, "usd", c)` emits `namespace` and `usd` as first-class JSON keys, so Loki/Elasticsearch can filter and aggregate `namespace="team-a"` across all 40 clusters without brittle regex on free-form strings. `fmt.Sprintf("scrape done namespace=%s", ns)` produces an opaque message you'd have to parse per-format at query time — it breaks the moment someone tweaks the wording, and you can't reliably `sum by (namespace)` over your logs.

**(c) How do you set an `http.Client` timeout and why must you?**
**Answer:** `client := &http.Client{Timeout: 5 * time.Second}` (optionally tighter per-request via `context.WithTimeout` + `http.NewRequestWithContext`). You must because Go's default client — and `http.Get` — has **no** timeout: a slow or hung upstream (a wedged cost API, a dead node) leaves the request blocked forever, goroutines and connections accumulate, and the exporter eventually OOMs or stops scraping. A bounded timeout turns a hang into a fast, logged error.

## Resources

1. **prometheus/client_golang** — [repo](https://github.com/prometheus/client_golang) · [pkg docs](https://pkg.go.dev/github.com/prometheus/client_golang/prometheus) — the exact `promauto`/`promhttp`/`GaugeVec` API you're using; the `prometheus` package overview plus `examples/` in the repo are the canonical exporter template. **Deep** (you'll reference it constantly building the exporter and later instrumenting the controller).
2. **`log/slog`** — [pkg docs](https://pkg.go.dev/log/slog) · [Go blog: structured logging](https://go.dev/blog/slog) — handlers, levels, `With`, attribute grouping; the blog explains the design and the `Handler` interface. **Skim the blog, deep on the pkg docs** for handler options.
3. **`net/http` & `encoding/json`** — [net/http](https://pkg.go.dev/net/http) · [encoding/json](https://pkg.go.dev/encoding/json) — `Server`/`Client` fields (timeouts!) and struct-tag/streaming semantics. **Skim now, deep on `Client`, `Server`, and `Decoder` when you hit a gotcha.**
4. **spf13/cobra** — [repo](https://github.com/spf13/cobra) — only if you go the subcommand route; the README's user-guide covers command trees and flag binding. **Skim** (optional for a single-command tool).
