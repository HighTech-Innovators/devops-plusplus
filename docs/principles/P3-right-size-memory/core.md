# P3 — Right‑Size Memory (Generic Core)

> **Principle:** Use only as much memory as needed for the job at hand, and release it promptly. Favor streaming, bounded buffers, and data pruning at the source so memory use scales with *value*, not with worst‑case inputs.

## Why it matters
- **Stability:** Fewer OOMs and GC storms, smoother tail latency.
- **Cost/Energy:** Smaller working sets → denser packing → lower watts per user action.
- **Performance:** Streaming avoids large pauses and enables parallel I/O.

## Signals & Targets (tune per team)
- **Memory per action** (MPA): `rss_delta / action` or `heap_inuse / request` — trend **↓**.
- **Working set vs limit:** keep steady‑state **≤ 60–70%** of limit; short spikes allowed.
- **Leak slope:** long‑window RSS/heap trend **≈ 0** under constant load (no creep).
- **I/O amplification:** prefer chunked/streamed reads/writes; avoid `SELECT *`/full loads.

## Design Patterns (Do This)
- **Stream large payloads** (HTTP, files, DB) — process in chunks; avoid building whole objects in memory.
- **Bound parsing and queues** — set max body size, max batch, max inflight work.
- **Prune at source** — project columns, filter rows, compress on the edge, not in the hot path.
- **Pool wisely** — reuse buffers/objects where safe; cap pool sizes; avoid unbounded caches.
- **Backpressure** — fail fast or shed load when queues fill; don’t buffer endlessly.
- **Right limits** — container memory limit, GC thresholds, and request size caps set intentionally.
- **Instrumentation** — export `heap_inuse`, `rss`, and queue depths; tie to endpoints/jobs.

## Anti‑Patterns (Don’t Do This)
- Loading entire files/responses into memory when a stream suffices.
- Unbounded caches/queues, `SELECT *`, or per-request `.toArray()` on huge cursors.
- Global compression/serialization in hot paths for large objects.
- Ignoring memory limits, relying on OOM killer as a “scale” signal.
- Silent leaks: event listeners/closures retaining large graphs.

## Observability (neutral examples)
- **Memory per request:** `delta(container_memory_working_set_bytes) / delta(http_requests_total)`
- **Heap in use:** runtime‑specific heap metrics (V8, JVM, Go) with p95/p99 panels.
- **Leak detection:** slope(RSS) over long windows; GC time vs allocation rate.

## CI/CD Guardrails
- ❌ p95 **memory per request** regression >10% vs main → **fail**.
- ❌ Long‑window **RSS slope > 0** under constant load → **fail**.
- ❌ New large file/DB operations without streaming → **fail**.
- ✅ PR auto‑comment: projected mWh delta from memory reduction (lower GC/IO).

## Acceptance Criteria
- Bounded request/body sizes, queues, and batch sizes documented and enforced.
- Streaming implemented for large payloads.
- Dashboards show MPA, heap/RSS, and GC time; no long‑term creep at steady load.