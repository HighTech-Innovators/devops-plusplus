---
principle: P3-right-size-memory
doc_type: best-practices
tags: [memory, efficiency, streaming, backpressure, observability, express, node]
---

# Right‑Size Memory — Best Practices (Generic + Express)

## Do this

- **Bound request bodies.** Enforce sane defaults and reject oversize inputs.
  - Express: `app.use(express.json({ limit: '1mb' }));`
- **Stream I/O.** Pipe files/HTTP/database rows instead of materializing.
  - Express: `req.pipe(fs.createWriteStream(path))`; `stream.pipeline(...)` with backpressure.
- **Project & filter early.** Push predicates to DB; avoid `SELECT *` and per‑request `.toArray()` on large cursors.
- **Chunk responses.** Use chunked transfer or pagination; avoid sending giant JSON arrays.
- **Cap concurrency & queues.** Use a limiter for inflight work; reject when saturated.
- **Use bounded caches.** LRU with max entries/bytes; TTLs tied to producer freshness.
- **GC hygiene & limits.** Set container memory limits; tune runtime flags where applicable.
- **Measure MPA (Memory per Action).** Emit `rss_delta` and `heap_inuse` per route; gate in CI.
- **Detect leaks fast.** Track long‑window RSS slope; add heap snapshots on thresholds.

### Express/Node patterns

**1) Bounded JSON & file uploads**

```js
const express = require('express');
const multer  = require('multer'); // for multipart/form-data
const upload = multer({ limits: { fileSize: 5 * 1024 * 1024 } }); // 5MB

app.use(express.json({ limit: '1mb' })); // bound JSON

app.post('/upload', upload.single('file'), (req, res) => {
  // Multer streams to tmp; enforce size limits
  res.status(201).json({ ok: true });
});
```

**2) Streaming proxy (no full buffering)**

```js
const { pipeline } = require('stream');
const http = require('http');

app.get('/proxy', (req, res) => {
  http.get('http://upstream/large', (up) => {
    pipeline(up, res, (err) => { if (err) req.log?.error(err); });
  });
});
```

**3) Limit inflight work**

```js
const pLimit = require('p-limit');
const limit = pLimit(Number(process.env.MAX_INFLIGHT || 50));

app.post('/process', async (req, res) => {
  await limit(async () => { /* do bounded work */ });
  res.status(202).end();
});
```

**4) Bounded caches**

```js
const LRU = require('lru-cache');
const cache = new LRU({ max: 5000, maxSize: 50*1024*1024, sizeCalculation: v => Buffer.byteLength(JSON.stringify(v)) });
```

**5) Memory per request metric**

```js
const onFinished = require('on-finished');
app.use((req, res, next) => {
  const rss0 = process.memoryUsage().rss;
  onFinished(res, () => {
    const rss1 = process.memoryUsage().rss;
    console.log(JSON.stringify({ event: 'memory_per_request', route: req.route?.path, rss_delta: rss1 - rss0 }));
  });
  next();
});
```

### CI/Perf checks
- Synthetic run of large inputs uses **streaming** and keeps **MPA ≤ baseline**.
- p95 **rss_delta** non‑regression gate; fail on >10% increase.
- Long‑run test shows **RSS slope ≈ 0**.

### Dashboards (PromQL‑ish)
- `rate(memory_per_request_rss_delta_bytes_sum[5m]) / rate(http_requests_total[5m])`
- `container_memory_working_set_bytes` with p95/p99 & leak slope.