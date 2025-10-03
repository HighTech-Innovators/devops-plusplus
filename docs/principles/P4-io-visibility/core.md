# P4 — I/O Visibility (Generic Core)

> **Principle:** Make I/O **measurable, attributable, and controllable**. You can’t optimize what you can’t see. Instrument bytes, ops, latency, errors, and **which dependency** you paid them to.

## Why it matters
- **Performance:** Most tail latency lives in I/O waits, not CPU.
- **Cost & Energy:** Over‑fetching and chatty I/O waste bandwidth, storage, and watts per user action.
- **Reliability:** Without timeouts/backoff/circuit‑breakers, I/O failures cascade.

## What to measure (per dependency)
- **Bytes:** `io_read_bytes_total`, `io_write_bytes_total`
- **Ops:** `io_ops_total` (read/write/connect/open/close)
- **Latency:** histograms `io_latency_seconds_bucket` (by op + dependency)
- **Errors:** `io_error_total{reason}` (DNS, TLS, timeout, 5xx, throttling)
- **Utilization per action:** `bytes_per_request`, `ops_per_request`
- **Concurrency & queueing:** inflight, queued, retries
- **Cache hit‑rate** for read‑heavy flows

> Label everything by **dependency** (e.g., `upstream=payments`, `db=orders`, `bucket=media`), **tenant/feature**, and **route/job**.

## Targets (tune per team)
- **Bytes per action** ↓ over time; set hard caps for critical paths.
- **p95 I/O latency** within SLO; spikes trigger backpressure/circuit breaker.
- **Retry rate** ≤ **1–3%**; reasons labeled and actionable.
- **Cache hit‑rate** ≥ **80%** for intended hot keys.

## Patterns (Do This)
- **Time‑boxed I/O**: deadlines + timeouts everywhere; propagate cancellation.
- **Backoff + jitter** for transient errors; circuit‑breakers for persistent failure.
- **Batch/coalesce** to reduce N+1/chatty calls; single‑flight for stampedes.
- **Streaming** large payloads; avoid full buffering.
- **Connection reuse/pooling** with per‑host concurrency caps.
- **Result shaping**: project columns, filter rows; compress at the edge, not the app hot path.
- **Expose dependency budgets** per route/job; abort early beyond limits.

## Anti‑Patterns (Don’t Do This)
- No timeouts; infinite reads/writes.
- N+1 queries or one request → many chats to the same dependency.
- Full buffering of large objects.
- Unbounded retries; sync logging on hot paths.
- Per‑request new TCP/TLS sessions; disabled keep‑alive w/o reason.
- Invisible caches with unknown hit‑rates.

## Observability (PromQL‑ish examples)
```
# Bytes per request by route
sum by (route)(rate(io_write_bytes_total{surface="web"}[5m]) + rate(io_read_bytes_total{surface="web"}[5m]))
  / sum by (route)(rate(http_requests_total{surface="web"}[5m]))

# I/O latency SLO per dependency
histogram_quantile(0.95, sum by (le, dependency)(rate(io_latency_seconds_bucket[5m])))

# Retry/error rate share
sum by (dependency)(rate(io_error_total[5m])) / sum by (dependency)(rate(io_ops_total[5m]))
```

## CI/CD Guardrails
- ❌ **Bytes/req** regression > **10%** on critical routes → **fail**.
- ❌ Missing **timeouts** in new I/O code → **fail** (lint rule).
- ❌ New fan‑out w/o batch/single‑flight plan → **fail**.
- ✅ PR auto‑comment: top dependencies by bytes/req and p95 I/O latency delta.

## Acceptance Criteria
- Standard middleware/helpers emit **I/O metrics** per dependency.
- Dashboards by **route/job** and **dependency** show bytes/ops/latency/errors.
- Timeouts, backoff, and circuit‑breakers configured and tested.