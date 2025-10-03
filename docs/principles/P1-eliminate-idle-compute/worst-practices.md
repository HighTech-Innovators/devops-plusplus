
---
principle: P1-eliminate-idle-compute
stack: express
language: javascript
doc_type: worst-practices
tags: [anti-patterns, idle-compute, node, web]
---

# Eliminate Idle Compute — Worst Practices (Express/Node)

Below are common anti‑patterns that create **idle churn** (CPU burn when no one benefits) or make it hard for the app to rest when there’s no work.

## Avoid this

### 1) Busy polling instead of events/webhooks
- **Smell:** Loops that check “are we done yet?” every X seconds.
- **Why it’s bad:** Constant timers wake the event loop and prevent deep idleness.
- **Example (bad):**
```js
// BAD: Polling status every 2s forever.
setInterval(async () => {
  const done = await checkStatus();
  if (done) console.log('done');
}, 2000);
```
- **Prefer:** Webhooks or push; if polling is unavoidable, exponential backoff + jitter + hard stop.

### 2) Global compression on the app tier
- **Smell:** `app.use(compression())` applied to every route.
- **Why it’s bad:** CPU spikes on each response; keeps cores active even at low traffic.
- **Prefer:** Do compression at the CDN/reverse‑proxy; only enable per‑route if strictly needed.

### 3) Chatty, synchronous logging on hot paths
- **Smell:** `console.log` on every request or synchronous file writes.
- **Why it’s bad:** Forces I/O and CPU during idle‑heavy periods; increases latency.
- **Prefer:** Structured metrics + sampled logs (1–10%); async exporters.

### 4) Unbounded body parsing / heavy middleware everywhere
- **Smell:** `express.json()` with no limit; giant parsers enabled globally.
- **Why it’s bad:** Large payloads dominate CPU and GC; idle windows vanish.
- **Prefer:** `express.json({ limit: '1mb' })`; mount heavy middleware per‑route.

### 5) No graceful shutdown; keep‑alives keep the process alive
- **Smell:** Never calling `server.close()`; infinite `keepAliveTimeout`.
- **Why it’s bad:** Idle connections prevent process exit; platform can’t scale to zero.
- **Prefer:** Explicit `keepAliveTimeout`/`headersTimeout`; shutdown after inactivity.

### 6) Permanent worker pools for rare CPU tasks
- **Smell:** Pre‑spawning N workers on boot for tasks that happen once in a while.
- **Why it’s bad:** Baseline CPU/memory burn; no true idle.
- **Prefer:** Spawn workers **on demand**; terminate when done.

### 7) Periodic “warmers” with no KPI
- **Smell:** Cron hits endpoints every minute “just to be fast”.
- **Why it’s bad:** Manufactures load; masks real cold‑start needs; wastes energy.
- **Prefer:** Only warm with a business KPI; otherwise accept cold‑starts or schedule narrowly.

### 8) Poll‑driven health probes that trigger work
- **Smell:** `/healthz` computes DB joins or cache rebuilds.
- **Why it’s bad:** Liveness probes become covert traffic; kills idle windows.
- **Prefer:** O(1) health checks; keep heavy checks on a separate `/readyz` behind sampling/guards.

### 9) Zombie timers and intervals with no backoff
- **Smell:** `setInterval(..., 1000)` everywhere.
- **Why it’s bad:** Continuous wakeups; cumulative CPU even when “idle”.
- **Prefer:** Event‑driven; or `setTimeout` with **backoff + jitter** and a **max duty‑cycle**.

### 10) Overly long keep‑alive; stale sockets
- **Smell:** 2–5 minute keep‑alives by default.
- **Why it’s bad:** Keeps the event loop active and memory pinned.
- **Prefer:** Reasonable timeouts (e.g., 5s keep‑alive, 10s headers) tuned to client behavior.

## Code smells (quick sniff test)

```js
// BAD: Global compression and sync logging on every request.
const compression = require('compression');
app.use(compression()); // ❌ global
app.use((req, res, next) => { console.log(req.method, req.url); next(); }); // ❌ chatty sync logs
app.use(express.json()); // ❌ no limit
setInterval(() => checkQueue(), 1000); // ❌ zombie polling
```

## Safer defaults (what “good” looks like)

```js
// GOOD: Per-route compression (rare), sampled logs, bounded parsing, event-driven.
const http = require('http');
const onFinished = require('on-finished');
app.use(express.json({ limit: '1mb' }));
const SAMPLE = Number(process.env.LOG_SAMPLE || 0.1);
app.use((req, res, next) => { if (Math.random() < SAMPLE) console.log('hit', req.method, req.url); next(); });

const server = http.createServer(app);
server.keepAliveTimeout = 5000;
server.headersTimeout  = 10000;

let idleTimer;
const INACTIVITY_MS = 10 * 60 * 1000;
app.use((req, res, next) => { onFinished(res, () => resetIdle()); next(); });
function resetIdle() { clearTimeout(idleTimer); idleTimer = setTimeout(() => server.close(() => process.exit(0)), INACTIVITY_MS); }
```

## Smells checklist

- [ ] Busy polling for statuses instead of webhooks/events
- [ ] Global compression at app tier
- [ ] Sync or chatty logging on hot paths
- [ ] Unbounded body parsing or heavy global middleware
- [ ] No graceful shutdown, no explicit keep‑alive timeouts
- [ ] Permanent worker pools for rare tasks
- [ ] Warmers with no KPI
- [ ] Health checks that trigger heavy work
- [ ] Intervals with no backoff/jitter/duty‑cycle
- [ ] Excessively long keep‑alive leading to stale sockets
