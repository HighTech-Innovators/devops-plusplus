
---
principle: P1-eliminate-idle-compute
stack: express
language: javascript
doc_type: best-practices
tags: [idle-compute, energy, ops, node, web]
---

# Eliminate Idle Compute — Best Practices (Express/Node)

_In an Express/Node app, “idle” means the process consumes near‑zero CPU when there’s nothing to do and, ideally, doesn’t run at all unless real work arrives._

## Do this

- **Prefer events/webhooks over polling.** Receive async completions on `/webhooks/*`; do **not** poll job status.
  - Example: background image job posts to `POST /webhooks/job-complete` when done.
- **Offload CPU-heavy work** (compression, image/video transforms, report generation) to **workers** or an external service.
  - Use `worker_threads`/queues so the web server stays responsive and sleepy between requests.
- **Bound request bodies & parsing.** Avoid accidental CPU and GC churn when idle periods break.
  - `app.use(express.json({ limit: '1mb' }));`
- **Keep-alive tuned, not infinite.** Prevent zombie sockets from waking the event loop.
  - `server.keepAliveTimeout = 5000; server.headersTimeout = 10000;`
- **Graceful idle shutdown after sustained inactivity.** Close the server and let the platform cold‑start on demand.
  - Use an inactivity timer reset on **response completion** (use `on-finished`), not merely on request arrival.
- **Lazy-load heavy modules** and avoid global compression/logging on the app tier.
  - Push compression to CDN/reverse-proxy; sample logs/metrics.
- **Sampled telemetry, async logging.** Emit metrics; log at ~1–10% sampling in prod to keep hot paths cheap.
- **Debounce/Single‑flight cache misses.** Coalesce N identical upstream calls into 1.
- **Define background task budgets.** Periodic tasks must have **trigger, expected rate, CPU budget, backoff + jitter**.
- **Right-size instances & enable scale-to-zero.** Keep startup cheap (fast route registration, lazy requires).

### Minimal configuration snippet

```js
// app.js
const express = require('express');
const http = require('http');
const onFinished = require('on-finished');
const app = express();

app.use(express.json({ limit: '1mb' })); // bound payloads

// Reset idle on completed responses
const INACTIVITY_MS = Number(process.env.INACTIVITY_MS || 10 * 60 * 1000);
let idleTimer;
function resetIdle(reason) {
  clearTimeout(idleTimer);
  idleTimer = setTimeout(() => {
    console.log(`[idle] ${INACTIVITY_MS}ms with no work (${reason}). Shutting down.`);
    server.close(() => process.exit(0));
  }, INACTIVITY_MS);
}
app.use((req, res, next) => { onFinished(res, () => resetIdle('request-complete')); next(); });

app.get('/healthz', (r, s) => s.status(204).end());

const server = http.createServer(app);
server.keepAliveTimeout = 5000;
server.headersTimeout = 10000;

server.listen(process.env.PORT || 3000, () => resetIdle('server-start'));
```

### Offloading heavy work (pattern)

```js
// routes/images.js
app.post('/images/process', async (req, res) => {
  const { imageUrl, callbackUrl } = req.body || {};
  if (!imageUrl || !callbackUrl) return res.status(400).json({ error: 'imageUrl and callbackUrl required' });

  const { Worker } = require('worker_threads');
  const worker = new Worker(`
    const { parentPort } = require('worker_threads');
    // Replace with real processing (no busy loops); post result when done.
    setTimeout(() => parentPort.postMessage({ ok: true }), 100);
  `, { eval: true });

  worker.once('message', () => worker.terminate());
  res.status(202).json({ accepted: true }); // fast-ack, complete via webhook
});
```

### Metrics to watch

- **OPUA (Ops per User Action)** → proxy via `cpu_seconds / request` per route.
- **Background CPU duty‑cycle** → `background_cpu_seconds / minute` when no requests.
- **Event loop health** → `nodejs_eventloop_lag_seconds` low and stable when idle.
- **I/O amplification** → `io_ops / request`, `bytes / request`; prefer batching.
- **Keep‑alive wakeups** → low/no wakeups during quiescent periods.
- **Memory footprint** → `heap_inuse / request` stable; RSS doesn’t grow at idle.

#### PromQL‑ish examples

```promql
# CPU per request (label by route)
sum(rate(process_cpu_seconds_total{app="web"}[5m])) by (route)
  / sum(rate(http_requests_total{app="web"}[5m])) by (route)

# Background CPU duty-cycle (zero-load window)
sum by (pod)(rate(process_cpu_seconds_total{app="web"}[5m]))
  unless sum by (pod)(rate(http_requests_total{app="web"}[5m])) > 0

# Event loop wakeups (proxy)
rate(nodejs_eventloop_lag_seconds_count{app="web"}[5m])
```

### CI gates (hard stops)

- **OPUA non‑regression:** p95 `cpu_seconds / request` ↑ **>10%** vs `main` on the same dataset → **fail**.
- **Idle duty‑cycle cap:** background CPU at idle **>5%** for 5 consecutive minutes in perf tests → **fail**.
- **No unmanaged periodic tasks:** any new cron/background loop **must** declare trigger, expected rate, CPU budget, backoff/jitter → missing → **fail**.
- **Keep‑alive sanity:** `keepAliveTimeout` and `headersTimeout` must be set explicitly → not set → **fail**.
- **Compression policy:** global in‑app compression for all routes → **flag** (require justification to pass).

### Governance checklist (ship‑it)

- [ ] Webhooks/events replace polling where feasible.
- [ ] Workers/queues handle CPU‑heavy tasks (web stays calm at idle).
- [ ] Body size limits enforced; heavy modules lazy‑loaded.
- [ ] Keep‑alive tuned; no zombie sockets.
- [ ] Idle shutdown working; process exits or sleeps after inactivity.
- [ ] Telemetry sampled; logs not synchronous per request.
- [ ] OPUA + idle duty‑cycle dashboards exist; CI gates enforced.
