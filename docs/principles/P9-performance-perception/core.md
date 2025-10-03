# P9 — Performance Perception (Generic Core, **sub‑1s mandate**)

> **Principle:** Optimize what users actually **feel**. For user‑visible actions, **p95 must stay under 1 second end‑to‑end**. If real work takes longer, **return feedback within 1s** and finish asynchronously (workers/webhooks/streams).

## Why it matters
- **User outcomes:** Sub‑second responses preserve flow and conversion.
- **Efficient compute:** Work users can’t perceive is often waste—especially on hot paths.
- **Governance:** Budgets tied to perception anchor engineering to business impact.

## Human time thresholds → tightened to sub‑1s
- **~100 ms**: *instant*. Taps, small actions, optimistic acks.
- **≤ 250 ms**: *snappy*. Keypress/validation, simple queries.
- **≤ 500 ms**: *smooth*. Most API calls, navigations that render above‑the‑fold.
- **≤ 1,000 ms**: *flow preserved*. If you can’t finish by now, show durable progress and **offload the rest**.
> Anything that cannot meet **≤1 s p95** must provide **first feedback within 1 s** and shift remaining work off the interaction path.

## Key perception metrics (choose per surface)
- **Web (RUM):** **TTFB**, **LCP**, **INP**, **CLS**, **TTI** — targets tuned for ≤1 s perceived waits on key actions.
- **API:** **p95 end‑to‑end latency**, **jitter** (p99‑p50), **TTFB**.
- **Mobile/Client:** **time‑to‑first‑interaction**, **frame time** (jank %), **offline ready**.
- **Flow:** time from **intent → first feedback → completion**; count **blocking steps**.

## Patterns (Do This)
- **First feedback < 1 s:** optimistic UI, skeletons, streaming partials, `202 Accepted` + status link.
- **Stabilize visuals:** reserve space; avoid layout shifts that delay first paint.
- **Defer invisible work:** prefetch/prerender strategically; lazy‑load heavy code.
- **Batch & debounce:** one round‑trip per intent; cache close to users.
- **Protect interaction paths:** heavy work → workers/queues; keep handlers constant‑time.
- **Measure in the field:** RUM + server metrics; correlate perception to **business events**.
- **CI budgets:** enforce <1 s p95 and bounded jitter before merge.

## Anti‑Patterns (Don’t Do This)
- Ship features that **block >1 s** without fast feedback and offload.
- Layout jumps; long spinners; hidden busy loops that stall interactivity.
- Chatty validations; multiple sequential calls per tap.

## Observability (illustrative)
```
# API perceived latency p95 (must be ≤ 1 s on user-visible actions)
histogram_quantile(0.95, sum by (le, route)(rate(http_server_duration_seconds_bucket[5m])))

# Jitter (keep small to maintain perceived smoothness)
histogram_quantile(0.99, ...) - histogram_quantile(0.50, ...)

# RUM (export via web vitals)
lcp_ms{app="web"}; inp_ms{app="web"}; cls{app="web"}
```

## CI/CD Guardrails
- ❌ **p95 user‑visible action latency > 1,000 ms** → **fail**.
- ❌ **Jitter** (p99‑p50) ↑ > **20%** vs baseline → **fail**.
- ❌ **Web vitals** (LCP/INP/CLS) regressions over budget → **fail**.
- ✅ PR comment: p50/p95, jitter, TTFB/LCP deltas; link to traces & RUM panels.

## Acceptance Criteria
- Sub‑1 s p95 budget per critical action; fallbacks provide first feedback <1 s.
- Dashboards show latency & jitter by action; RUM and server metrics stitched.
- Async/offload for work that cannot complete inside 1 s.