---
principle: P6-prune-data
doc_type: best-practices
tags: [data-minimization, governance, efficiency, observability, express, node]
---

# Prune Data — Best Practices (Generic + Express)

## Do this

- **Collect less**: Only fields needed for the feature. Make new fields **opt‑in** with TTL and owner.
- **Filter at the source**: Apply WHERE predicates, CDC filters, or client‑side guards to avoid ingesting junk.
- **Project columns**: Return the minimal set the UI actually renders.
- **Paginate/stream** large results: never dump thousands of rows in one go.
- **Summarize for exploration**: sketches/aggregates for dashboards; compute exact values on commit paths.
- **Enforce TTL & retention**: auto-expire logs/events; store PII only as long as necessary.
- **Deduplicate by key**: idempotent writes; compaction jobs; avoid duplicate fan‑out.
- **Measure selectivity & utilization**: emit metrics for scanned vs used, field read rates, TTL success.
- **Schema hygiene**: deprecate unused fields; version responses; remove dead params.

### Express/Node examples

**1) Project fields to what the client uses**

```js
// Controller only returns needed fields
app.get('/orders/:id', async (req, res) => {
  const row = await db.query('SELECT order_id, total, status FROM orders WHERE order_id=$1', [req.params.id]);
  res.json({ id: row.order_id, total: row.total, status: row.status });
});
```

**2) Pagination & max page size**

```js
app.get('/orders', async (req, res) => {
  const page = Math.max(1, Number(req.query.page || 1));
  const size = Math.min(100, Number(req.query.size || 20)); // bound page size
  const rows = await db.query('SELECT order_id,total,status FROM orders ORDER BY created_at DESC LIMIT $1 OFFSET $2', [size, (page-1)*size]);
  res.json(rows);
});
```

**3) Field utilization metric**

```js
// Track which response fields are actually read by the client (reported back from frontend)
app.post('/telemetry/field-read', (req, res) => {
  const { field } = req.body || {};
  if (field) console.log(JSON.stringify({ event: 'field_read', field }));
  res.status(204).end();
});
```

**4) TTL and deletion proof (logs)**

```js
// Example: delete expired sessions
await db.query('DELETE FROM sessions WHERE expires_at < NOW() RETURNING 1');
console.log(JSON.stringify({ event: 'ttl_expire_run', deleted: /* count */ 42 }));
```

**5) Deduplicate inputs**

```js
await db.query(`
  INSERT INTO events (tenant, event_id, payload)
  VALUES ($1,$2,$3)
  ON CONFLICT (tenant, event_id) DO NOTHING
`, [tenant, eventId, payload]);
```

### Dashboards
- **Bytes/req** by route; **records scanned vs used**; **field utilization** top/bottom.
- **TTL success**; **duplication factor** over time; **payload size** distributions.

### CI gates
- Fail PRs adding fields without **owner + TTL + deprecation plan**.
- Fail on **bytes/req** regression >10% for protected routes.
- Flag responses with **> N fields** or **> max payload** without pagination.