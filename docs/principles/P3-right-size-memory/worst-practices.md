---
principle: P3-right-size-memory
doc_type: worst-practices
tags: [anti-patterns, memory, leaks, buffering, express, node]
---

# Right‑Size Memory — Worst Practices (Generic + Express)

## Avoid this

- **Reading whole files/bodies into memory** when streams suffice.
- **Unbounded caches/queues** that grow with traffic or input size.
- **`SELECT *` and `cursor.toArray()`** on hot paths; materializing large result sets.
- **Global compression/serialization** of large payloads in the app tier.
- **Silent leaks** via event listeners/closures retaining buffers.
- **Ignoring limits** (container, request sizes) and relying on OOM killer.

### Express/Node anti‑patterns & fixes

**1) Full buffering (bad)**
```js
// BAD: collects entire upstream into a string
app.get('/big', async (req, res) => {
  const resp = await fetch('http://upstream/huge.json');
  const text = await resp.text(); // ❌ huge string in memory
  res.type('json').send(text);
});
```
**Fix — stream**

```js
app.get('/big', async (req, res) => {
  const resp = await fetch('http://upstream/huge.json');
  resp.body.pipe(res); // ✅ stream without buffering whole body
});
```

**2) Unbounded body parsing (bad)**
```js
app.use(express.json()); // ❌ no limit
```
**Fix**
```js
app.use(express.json({ limit: '1mb' })); // ✅
```

**3) Unbounded cache (bad)**
```js
const cache = new Map(); // ❌ grows forever
```
**Fix**
```js
const LRU = require('lru-cache');
const cache = new LRU({ max: 5000, ttl: 60_000 });
```

**4) Wide DB fetch (bad)**
```js
const rows = await db.collection('orders').find({}).toArray(); // ❌ materializes all
```
**Fix**
```js
const cursor = db.collection('orders').find({ status: 'pending' }).project({ order_id: 1, amount: 1 });
for await (const doc of cursor) { /* stream rows */ }
```

**5) Leak via listeners (bad)**
```js
app.use((req, res, next) => {
  const buf = Buffer.alloc(1024*1024);
  res.on('close', () => console.log('done')); // ❌ buf closed over & retained if not careful
  next();
});
```
**Fix**
```js
app.use((req, res, next) => {
  let buf = Buffer.allocUnsafe(1024); // smaller, recycled
  res.once('close', () => { buf = null; }); // release
  next();
});
```

### Smells checklist
- [ ] p95 `rss_delta` rising >10% vs baseline.
- [ ] Long‑run **RSS slope > 0** under constant load.
- [ ] `.toArray()` on hot paths; wide scans without projection.
- [ ] Global compression/serialization on large payload routes.
- [ ] `express.json()` without a **limit**.
- [ ] Caches/queues with no **max**/**TTL**.