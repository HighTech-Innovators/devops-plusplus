---
principle: P10-efficiency-user-terms
doc_type: worst-practices
tags: [anti-patterns, user-terms, value, cost, energy, express, node]
---

# Efficiency In User Terms — Worst Practices (Generic + Express)

## Avoid this

- **Engineering‑only metrics**: celebrating QPS/CPU% without **€/value**, **mWh/1k**, or **TTV**.
- **No Value Events**: features ship without observable outcomes to divide by.
- **Averages hide pain**: no per‑tenant/region/experiment breakdowns; hotspots invisible.
- **Throughput worship**: higher QPS but worse **TTV** and higher **cpu/bytes per value**.
- **Free features** that cost real money/energy per use and deliver little or no value.
- **No PR accounting**: merges land without efficiency impact called out.

### Express/Node anti‑patterns & fixes

**1) “It’s faster” without value (bad)**
```md
We improved QPS by 30%.
```
**Fix — convert to user terms**
```md
TTV p95 improved from 1.2 s → 0.9 s (flow preserved); cpu/value -18%; €/order -12%.
```

**2) No Value Event (bad)**
```js
app.post('/feature', (req, res) => { /* returns 200 */ });
```
**Fix — emit and attribute**
```js
app.post('/feature', valueEvent('feature_used', { tenant, region }), handler);
```

**3) Global averages (bad)**
- EU tenants suffer; average looks fine.

**Fix — slice**
- Per‑tenant/region dashboards; alert on local regressions.

**4) Shipping costly freebie (bad)**
- Heavy export/AI call free to users but € and mWh per use high.

**Fix — show €/value + mWh/1k** to product; add quotas or price signals; optimize or cut.

### Smells checklist
- [ ] No **€/value** or **mWh/1k** in dashboards.
- [ ] No **TTV** budget; p95 > 1 s on key actions.
- [ ] New feature without **Value Event** instrumentation.
- [ ] CPU/bytes/value trending ↑ over releases.
- [ ] Global metrics with no tenant/region breakdown.