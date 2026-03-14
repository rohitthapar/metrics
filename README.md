# gomet — Go Metrics SDK

A lightweight, embeddable metrics SDK for Go applications. Emit counters, gauges, timers, and custom events from anywhere in your code and visualize them on a local real-time dashboard — no external services required.

---

## Features

- **Counter** — increment/decrement named counters with tags
- **Gauge** — set arbitrary numeric values (memory, queue depth, etc.)
- **Timer** — measure and record latency with `Start()`/`Stop()`
- **Event** — emit custom events with arbitrary metadata
- **In-process buffer** — non-blocking hot path; batches and retries before flushing
- **Local dashboard** — real-time charts and event feed served at `localhost:9091`
- **Zero dependencies** — single binary, no external databases or brokers

---

## Architecture

```
Your Application
└── SDK (github.com/you/gomet/sdk)
    ├── Counter / Gauge / Timer / Event
    └── In-process buffer (batches + retries)
            │
            │  HTTP POST /metrics  (batch flush)
            ▼
    Collector Server  :9090
    ├── Ingest API       POST /metrics
    ├── Aggregator       roll-up, flush intervals
    ├── Storage          in-memory ring buffer
    └── Query API        GET /query + WebSocket /ws
            │
            │  WebSocket (live) + HTTP (historical)
            ▼
    Web Dashboard  :9091
    ├── Timeseries charts
    ├── Live event feed
    └── Filter by name, tag, time range
```

---

## Project Structure

```
gomet/
├── sdk/
│   ├── client.go        # Client struct, New(), options
│   ├── counter.go       # Counter metric
│   ├── gauge.go         # Gauge metric
│   ├── timer.go         # Timer / latency
│   ├── event.go         # Custom events with metadata
│   └── buffer.go        # In-process batching + retry
├── server/
│   ├── main.go          # Starts collector + dashboard servers
│   ├── ingest.go        # POST /metrics handler
│   ├── query.go         # GET /query, WebSocket /ws
│   ├── store.go         # In-memory ring-buffer storage
│   └── dashboard/
│       └── index.html   # Embedded UI (go:embed)
└── examples/
    └── demo/main.go
```

---

## Installation

```bash
go get github.com/you/gomet/sdk
```

Start the collector + dashboard server (ships as a standalone binary):

```bash
go install github.com/you/gomet/server@latest
gomet-server
# Collector listening on :9090
# Dashboard available at http://localhost:9091
```

---

## Quick Start

```go
package main

import (
    "time"
    "github.com/you/gomet/sdk"
)

func main() {
    m := sdk.New("http://localhost:9090",
        sdk.WithFlushInterval(5*time.Second),
        sdk.WithBatchSize(100),
    )
    defer m.Flush() // flush remaining metrics on exit

    // Counter
    m.Counter("api.requests").Inc(map[string]string{"route": "/login"})

    // Gauge
    m.Gauge("memory.mb").Set(512.4)

    // Timer
    t := m.Timer("db.query").Start()
    // ... do work ...
    t.Stop() // records latency automatically

    // Event with metadata
    m.Event("user.signup", map[string]any{
        "plan":   "pro",
        "region": "us-east",
    })
}
```

Open `http://localhost:9091` to see your metrics in real time.

---

## SDK Reference

### Initializing the client

```go
m := sdk.New(collectorURL string, opts ...Option)
```

| Option | Default | Description |
|---|---|---|
| `WithFlushInterval(d)` | `5s` | How often the buffer flushes to the server |
| `WithBatchSize(n)` | `100` | Max metrics per flush batch |
| `WithRetry(n)` | `3` | Number of retry attempts on failed flush |
| `WithTimeout(d)` | `3s` | HTTP request timeout |

### Counter

```go
c := m.Counter("name")
c.Inc(tags)          // +1
c.Add(n, tags)       // +n
c.Dec(tags)          // -1
```

### Gauge

```go
g := m.Gauge("name")
g.Set(value, tags)   // set to value
g.Inc(tags)          // +1
g.Dec(tags)          // -1
```

### Timer

```go
t := m.Timer("name").Start()
// ... work ...
duration := t.Stop()   // records latency, returns time.Duration
```

Or record a duration directly:

```go
m.Timer("name").Record(duration, tags)
```

### Event

```go
m.Event("event.name", map[string]any{
    "key": "value",
    // any JSON-serialisable fields
})
```

### Tags

Tags are `map[string]string` and can be attached to any metric. They appear as filterable dimensions in the dashboard.

```go
tags := map[string]string{
    "env":     "production",
    "service": "auth",
}
m.Counter("requests").Inc(tags)
```

---

## Dashboard

The dashboard is served as an embedded single-page app — no separate frontend build needed.

| URL | Description |
|---|---|
| `http://localhost:9091` | Main dashboard |
| `http://localhost:9090/metrics` | Raw ingest endpoint (POST) |
| `http://localhost:9090/query` | Query API (GET) |
| `ws://localhost:9091/ws` | Live WebSocket feed |

**Features:**
- Timeseries charts per metric name
- Real-time event log with metadata viewer
- Filter by metric name, tag key/value, and time range
- Drill-down from chart to raw data points

---

## How the buffer works

The `m.Counter(...).Inc(...)` call is non-blocking — it appends to an internal channel immediately and returns. A background goroutine reads from the channel and accumulates metrics into a batch. The batch is flushed to the collector when either the flush interval elapses or the batch reaches its size limit, whichever comes first.

On failed flush, the batch is retried up to `WithRetry(n)` times with exponential backoff. If retries are exhausted, the batch is dropped and a warning is logged.

Call `m.Flush()` before your process exits to drain any remaining metrics:

```go
defer m.Flush()
```

---

## Storage

The collector uses an in-memory ring buffer by default — fast and zero-config, but data does not survive a server restart.

To persist metrics across restarts, set the storage path flag when starting the server:

```bash
gomet-server --storage ./metrics.db
```

This switches to a file-backed store (SQLite via `modernc.org/sqlite`).

---

## Build Phases

| Phase | Scope |
|---|---|
| 1 — SDK core | `Counter`, `Gauge`, `Timer`, `Event`, in-process buffer, HTTP flush |
| 2 — Collector server | Ingest handler, in-memory store, query API |
| 3 — Dashboard | Embedded HTML, Chart.js timeseries, WebSocket live tail, tag filters |
| 4 — Polish | Retry logic, auth token, UDP mode, graceful shutdown, `go:embed` assets |

---

## Contributing

```bash
git clone https://github.com/you/gomet
cd gomet
go test ./...
```

Please open an issue before submitting a large PR.

---

## License

MIT