---
principle: P9-performance-perception
doc_type: best-practices
tags: [perception, sub-1s, rum, web-vitals, latency, jitter, express, node]
---

# Performance Perception — Best Practices (**keep ≤ 1 s**)

## Do this

- **Set hard budgets:** user-visible actions **p95 ≤ 1,000 ms**. Prefer **≤100 ms** instant acks; **≤250 ms** validation; **≤500 ms** above‑the‑fold.
- **Stream first bytes fast:** send headers/snippets within **≤250 ms**; render shell/skeleton immediately.
- **Optimistic UI:** acknowledge within **≤100 ms**; reconcile when server confirms.
- **Offload long work:** return **202** within **≤1 s**; complete via worker + webhook/notifications.
- **Debounce & batch:** 1 round‑trip per intent; collapse chatty validations.
- **Edge caching:** ETags/conditional requests; keep TTFB low.
- **Protect handlers:** constant‑time checks; heavy compute off hot path.
- **Measure & alert:** RUM vitals + API p95/jitter; alert when **p95 > 1 s** or **V/CPU‑s drops**.

### Express/Node patterns (sub‑1s)

**1) First paint/TTFB < 250 ms: flush early**
```js
app.get('/page', async (req, res) => {
  res.setHeader('Content-Type', 'text/html; charset=utf-8');
  res.write('<!doctype html><html><head><title>Fast</title></head><body>');
  res.write('<div id="shell"><h1>Hi!</h1></div>'); // above-the-fold
  res.flushHeaders?.(); // send early within ~250 ms
  const below = fetchBelow().catch(()=>'<p>later</p>');
  res.write(await below);
  res.end('</body></html>');
});
```

**2) API p95 ≤ 1 s: stream/chunk**
```js
app.get('/items', async (req, res) => {
  res.setHeader('Content-Type', 'application/json');
  res.write('[');
  let first = true;
  for await (const item of streamItems()) {
    if (!first) res.write(',');
    first = false;
    res.write(JSON.stringify(item)); // partial results keep perceived speed < 1 s
  }
  res.end(']');
});
```

**3) Long task → 202 within 1 s**
```js
app.post('/checkout', async (req, res) => {
  enqueue('charge', req.body);      // offload
  res.status(202).json({ accepted: true, status: '/status/...' }); // reply < 1 s
});
```

**4) RUM hook (Web Vitals)**
```js
app.post('/rum/vitals', express.json({limit:'50kb'}), (req, res) => {
  const { lcp_ms, inp_ms, cls } = req.body || {};
  console.log(JSON.stringify({ event: 'web_vital', lcp_ms, inp_ms, cls, route: req.headers['x-route'] }));
  res.status(204).end();
});
```

### Dashboards
- p50/p95 latency and **jitter** per action; **alerts when p95 > 1 s**.
- Web vitals stitched to routes/releases; conversion vs latency cohorts.

### CI checks
- Lighthouse/Playwright: fail if **LCP/INP/CLS** exceed budgets for key pages.
- Autocannon/k6: fail if **p95 > 1 s** or jitter regression > **20%**.