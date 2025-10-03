---
principle: P4-io-visibility
doc_type: best-practices
tags: [io, latency, observability, backpressure, batching, express, node]
---

# I/O Visibility — Best Practices (Generic + Express)

## Do this

- **Instrument bytes/ops/latency/errors** per dependency (DB, cache, HTTP upstreams, object store).
- **Set deadlines & timeouts** on every I/O path; propagate cancellation.
- **Batch/coalesce** (kill N+1); implement **single‑flight** for identical upstream fetches.
- **Stream large payloads** and enable backpressure via `stream.pipeline`.
- **Reuse connections** with pools/keep‑alive; cap **per‑host concurrency**.
- **Classify errors** with reason codes (DNS, timeout, throttled, 5xx).
- **Expose budgets**: bytes/req, ops/req ceilings per route/job.
- **Cache where it helps** and emit hit/miss; measure utilization, not just fills.
- **Log exemplars** that tie big I/O to traces for root‑cause.

### Express/Node patterns

**1) I/O metrics middleware (bytes/ops/latency)**

```js
// io-metrics.js
const onFinished = require('on-finished');
module.exports.ioMetrics = (req, res, next) => {
  const start = process.hrtime.bigint();
  const s = req.socket;
  const r0 = s.bytesRead, w0 = s.bytesWritten;
  onFinished(res, () => {
    const dur = Number(process.hrtime.bigint() - start) / 1e9;
    const read = s.bytesRead - r0, wrote = s.bytesWritten - w0;
    console.log(JSON.stringify({ event: 'io_summary', route: req.route?.path || req.path, read_bytes: read, wrote_bytes: wrote, latency_s: dur }));
  });
  next();
};
```

**2) HTTP client helper with timeout, retries, and labeling**

```js
// httpx.js
const http = require('http'); const https = require('https');
function requestWithTimeout(url, { timeoutMs = 5000, retries = 2, dependency = 'upstream' } = {}) {
  const lib = url.startsWith('https') ? https : http;
  return new Promise((resolve, reject) => {
    const req = lib.get(url, { timeout: timeoutMs, headers: { Connection: 'keep-alive' } }, (res) => resolve(res));
    req.on('timeout', () => { req.destroy(new Error('timeout')); });
    req.on('error', (e) => reject(e));
  }).catch(async (err) => {
    if (retries > 0 && /timeout|ECONNRESET|EAI_AGAIN/.test(String(err?.message))) {
      console.log(JSON.stringify({ event: 'io_retry', dependency, reason: String(err.message) }));
      return requestWithTimeout(url, { timeoutMs: timeoutMs * 1.5, retries: retries - 1, dependency });
    }
    console.log(JSON.stringify({ event: 'io_error', dependency, reason: String(err?.message) }));
    throw err;
  });
}
module.exports = { requestWithTimeout };
```

**3) Single‑flight fetch to avoid stampedes**

```js
// singleflight.js
const inflight = new Map();
async function singleFlight(key, fn) {
  if (inflight.has(key)) return inflight.get(key);
  const p = fn().finally(() => inflight.delete(key));
  inflight.set(key, p); return p;
}
module.exports = { singleFlight };
```

**4) Streaming proxy with backpressure**

```js
const { pipeline } = require('stream');
app.get('/big', async (req, res) => {
  const up = await requestWithTimeout('http://upstream/large', { dependency: 'large_api' });
  pipeline(up, res, (err) => { if (err) console.error('stream_error', err); });
});
```

**5) DB projection & batching (pseudo)**

```js
// Only needed columns; paginate/batch to cap memory & bytes
SELECT order_id, amount FROM orders WHERE status = 'pending' LIMIT 500;
```

### Dashboards (PromQL‑ish)
- **Bytes/req by route** and **p95 I/O latency by dependency**.
- **Retry/error rate** by reason; **cache hit‑rate** trends.
- **Top N routes** by bytes/req; **Top N dependencies** by latency.

### CI gates
- Fail PRs that add I/O without **timeouts** or **labels**.
- Fail on **bytes/req** regression >10% for protected routes.
- Lint for **N+1** risks (multiple similar calls in a loop) and missing **single‑flight** on cacheable fetches.