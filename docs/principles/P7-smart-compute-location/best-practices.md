---
principle: P7-smart-compute-location
doc_type: best-practices
tags: [compute-placement, edge, workers, pushdown, serverless, observability, express, node]
---

# Smart Compute Location — Best Practices (Generic + Express)

## Do this

- **Render/validate at the client** when safe; avoid shipping secrets; degrade gracefully offline.
- **Use edge for fast decisions**: auth, AB routing, cache keys, small transforms (headers, cookies, JSON shaping).
- **Keep front door thin**: orchestrate, validate, emit value events; avoid CPU/GPU heavy work.
- **Offload heavy tasks to workers**: image/video processing, PDFs, bulk exports, ML inference — event-driven.
- **Push computations to data**: DB WHERE/projection/aggregations; OLAP cubes; search facets; server-side cursors.
- **Adopt serverless for bursty jobs**: natural scale-to-zero; ensure cold-start budget is acceptable.
- **Cache where it pays**: edge/local caches with clear TTLs; high hit-rates for hot keys.
- **Respect regions**: process PII in-region; minimize cross-region egress; replicate selectively.
- **Measure surface split**: track latency/bytes/value by **client/edge/server/worker**; optimize where it stings.

### Express/Node examples

**1) Thin front door → worker handoff**

```js
// routes/report.js (front door)
app.post('/reports/export', async (req, res) => {
  // Validate and enqueue
  await queue.add('export_report', { userId: req.user.id, params: req.body });
  res.status(202).json({ accepted: true }); // UI polls via webhook or notifications
});
```

```js
// workers/export.js (heavy CPU/I/O off main web tier)
queue.process('export_report', async (job) => {
  // Generate PDF/CSV, store in object storage
  // POST webhook to notify completion
});
```

**2) Edge function for AB routing + cache key shaping (pseudo)**

```js
// edge/ab-routing.js
export default async function handle(request) {
  const variant = Math.random() < 0.5 ? 'A' : 'B';
  request.headers.set('x-ab', variant);
  // Choose cache key by tenant + path + variant
  return fetch(request);
}
```

**3) DB pushdown (projection/filter/aggregation)**

```sql
SELECT order_id, total
FROM orders
WHERE tenant = $1 AND created_at >= NOW() - interval '7 days'
ORDER BY created_at DESC
LIMIT 200;
```

**4) Serverless for bursty tasks**

```js
// cloud function triggered by object upload
exports.generateThumbnail = async (event) => { /* read small, write small; finish fast */ };
```

**5) Telemetry: split by surface**

```js
// Log labels that attribute value events and latency to client/edge/server/worker
console.log(JSON.stringify({ event: 'value', surface: 'server', route: '/checkout', latency_ms }));
```

### Dashboards
- Latency p95 by **surface** and route/job.
- Egress bytes per value event, by region.
- Worker queue age vs throughput; duty cycle & scale-to-zero windows.
- Pushdown ratio; cache hit-rate by edge/location.

### CI gates
- Fail PRs that add CPU-heavy code on the front door without worker/serverless offload.
- Fail cross-region calls w/o caching or replication rationale.
- Fail large queries without projections/filters; require pushdown proof.