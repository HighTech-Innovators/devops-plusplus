---
principle: P9-performance-perception
doc_type: worst-practices
tags: [anti-patterns, sub-1s, perception, rum, latency, jitter, express]
---

# Performance Perception — Worst Practices (violates **≤ 1 s**)

## Avoid this

- **Blocking > 1 s** on user-visible interactions without fast feedback/offload.
- **Slow first byte**: no early bytes/shell; users stare at a blank screen.
- **Chatty flows**: multiple sequential requests per intent.
- **Layout shifts**: late asset loads that delay usable paint.
- **Heavy compute on hot path**: long DB scans, PDF generation, image processing inline.
- **No RUM**: lab-only numbers; no field data to prove sub‑1 s.

### Express/Node anti‑patterns & fixes

**1) >1 s handler (bad)**
```js
app.post('/export', async (req, res) => {
  const pdf = await generateHugePDF(); // ❌ blocks user >1 s
  res.send(pdf);
});
```
**Fix — offload + 202 within 1 s**
```js
app.post('/export', (req, res) => { enqueue('export', req.body); res.status(202).json({ accepted:true }); });
```

**2) Slow first paint (bad)**
```js
// heavy DB fetch before writing any bytes
```
**Fix — flush shell in ≤250 ms, stream below-the-fold later.**

**3) Chatty validations (bad)**
```js
// multiple round-trips for field checks
```
**Fix — batch + debounce; pre-validate on client; cache hints.**

**4) Layout instability (bad)**
- Images without dimensions; components shift on load.

**Fix — reserve space; skeletons; lazy-load below the fold.**

**5) No budgets/alerts (bad)**
- No p95≤1 s budget or alerts; regressions slip into prod.

**Fix — enforce CI gates and RUM dashboards with p95/jitter panels.**

### Smells checklist
- [ ] Any user-visible action with **p95 > 1 s**.
- [ ] TTFB > 250 ms for key pages.
- [ ] Multiple sequential network trips per intent.
- [ ] LCP/INP/CLS regressions without rollback.
- [ ] No RUM telemetry stitched to routes/releases.