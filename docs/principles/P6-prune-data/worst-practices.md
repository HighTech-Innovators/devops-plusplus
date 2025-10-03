---
principle: P6-prune-data
doc_type: worst-practices
tags: [anti-patterns, data-bloat, governance, express, node]
---

# Prune Data — Worst Practices (Generic + Express)

## Avoid this

- **Kitchen‑sink responses**: returning every field “just in case.”
- **`SELECT *`** and materializing entire tables/collections.
- **No TTL/retention**: keeping PII/logs/events indefinitely.
- **Duplication**: recomputing/replicating the same data in multiple places without ownership.
- **Huge pages**: returning thousands of records; no pagination/streaming.
- **Silent schema sprawl**: adding fields without owners, TTLs, or deprecation process.

### Express/Node anti‑patterns & fixes

**1) SELECT * (bad)**
```js
const rows = await db.query('SELECT * FROM orders'); // ❌
```
**Fix**
```js
const rows = await db.query('SELECT order_id,total,status FROM orders WHERE tenant=$1 LIMIT 100', [tenant]);
```

**2) Kitchen‑sink response (bad)**
```js
res.json(row); // ❌ unfiltered DB row with dozens of fields
```
**Fix**
```js
res.json({ id: row.order_id, total: row.total, status: row.status }); // ✅ minimal projection
```

**3) No TTL/retention (bad)**
```js
// logs/events stored forever
```
**Fix**
```js
// set TTL at the store and verify deletion
await db.query('DELETE FROM logs WHERE created_at < NOW() - interval '30 days'');
```

**4) Duplicate pipelines (bad)**
```js
// two jobs compute the same aggregate in different repos
```
**Fix** — single owner; others subscribe; add dedupe keys.

**5) Giant pages (bad)**
```js
app.get('/orders', async (req, res) => { const rows = await db.query('SELECT ...'); res.json(rows); });
```
**Fix** — paginate; cap page size; stream when large.

### Smells checklist
- [ ] Bytes/req ↑ >10% vs baseline.
- [ ] `SELECT *` or unprojected responses in reviews.
- [ ] Fields added without owner/TTL/deprecation.
- [ ] Low field utilization (<10%) persisting beyond N weeks.
- [ ] High duplication factor across stores/pipelines.