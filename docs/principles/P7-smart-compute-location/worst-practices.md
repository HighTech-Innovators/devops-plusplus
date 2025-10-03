---
principle: P7-smart-compute-location
doc_type: worst-practices
tags: [anti-patterns, compute-placement, latency, egress, express, node]
---

# Smart Compute Location — Worst Practices (Generic + Express)

## Avoid this

- **Doing everything on the front door**: compression, image transforms, report generation, fan-out to many services.
- **Ignoring data gravity**: pulling huge datasets across regions/AZs instead of computing near the data.
- **No edge**: missing easy wins like auth hints, cache shaping, and A/B routing close to users.
- **Always-on workers** for rare jobs; no scale-to-zero; busy waiting on queues.
- **No pushdown**: `SELECT *` then filtering in app; aggregations outside OLAP/search that can do it faster/cheaper.
- **Cross-region chatty calls** without caching/replication; surprise egress bills.
- **PII processed out-of-region** without a compliance reason.

### Express/Node anti-patterns & fixes

**1) Heavy compute on web tier (bad)**
```js
app.post('/images/resize', async (req, res) => {
  // ❌ CPU-heavy resize here blocks event loop
});
```
**Fix — worker/serverless**
```js
// Enqueue, respond 202; worker does the resize and posts completion webhook
```

**2) No edge routing/caching (bad)**
- All requests hit origin; cache keys ignore tenant/locale/variant.

**Fix — edge key shaping**
- Add headers at the edge and key the cache by `tenant + path + variant`; serve from PoP.

**3) Cross-region fetch in hot path (bad)**
```js
await fetch('https://us-west.api/thing'); // ❌ high RTT + egress
```
**Fix — region-local**
- replicate read-mostly data or add a region-local cache; avoid per-request cross-region RTT.

**4) No pushdown (bad)**
```js
const rows = await db.query('SELECT * FROM orders'); // ❌ then filter in Node
```
**Fix — pushdown**
```sql
SELECT order_id, total FROM orders WHERE tenant=$1 AND status='paid' LIMIT 200;
```

**5) Always-on background pool (bad)**
```js
const pool = new WorkerPool({ size: 8 }); // ❌ idle burn for rare jobs
```
**Fix — on-demand**
- Spawn workers only on job arrival; use serverless if bursty.

### Smells checklist
- [ ] Front door doing CPU/GPU heavy work.
- [ ] High egress per action; cross-region calls in hot paths.
- [ ] No edge functions for cache shaping/A/B.
- [ ] Queries without pushdown on large datasets.
- [ ] Workers never scaling to zero; idle duty cycle > 5%.