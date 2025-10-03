---
principle: P10-efficiency-user-terms
doc_type: best-practices
tags: [user-terms, value, cost, energy, observability, express, node]
---

# Efficiency In User Terms — Best Practices (Generic + Express)

## Do this

- **Name the Value Events** per feature (`order_committed`, `message_delivered`); ensure metric + log + trace exist.
- **Publish RPV dashboards** per feature: CPU/bytes/memory/**mWh**/**€** per value. Trend ↓ and alert on spikes.
- **Write budgets in user terms**: “Checkout: **€0.001/order**, **mWh/1k ≤ 250**, **TTV p95 ≤ 1 s**.”
- **Design with RPV in mind**: batch, cache, pushdown, offload heavy work; kill features that burn without value.
- **Show cost in PRs**: add an “Impact” section with **RPV and TTV deltas**; block merges on regressions.
- **Scope by tenant/region/experiment**: attribute RPV to where value occurs; avoid averages hiding hotspots.
- **Convert tech changes to € and mWh**: make trade‑offs visible to product.

### Express/Node examples

**1) Emit Value Event + RPV atoms**

```js
// telemetry/value.js
const onFinished = require('on-finished');
module.exports.valueEvent = (name, labels = {}) => (req, res, next) => {
  const cpu0 = process.cpuUsage(); const rss0 = process.memoryUsage().rss;
  const t0 = process.hrtime.bigint();
  onFinished(res, () => {
    if (res.statusCode >= 200 && res.statusCode < 300) {
      const cpu = process.cpuUsage(cpu0);
      const rss = process.memoryUsage().rss - rss0;
      const durMs = Number(process.hrtime.bigint() - t0) / 1e6;
      console.log(JSON.stringify({ event: 'value_event', name, ...labels, usr_us: cpu.user, sys_us: cpu.system, rss_delta: rss, dur_ms: durMs }));
    }
  });
  next();
};
```

**2) Apply to a business route**

```js
app.post('/orders/commit', valueEvent('order_committed', { tenant: 'acme', region: 'eu-west' }), async (req, res) => {
  // ... idempotent persist
  res.status(201).json({ ok: true });
});
```

**3) Compute RPV in metrics pipeline (prom‑ish examples)**

```promql
cpu_per_value = sum by (feature)(rate(cpu_seconds_value_total[5m])) / sum by (feature)(rate(value_events_total[5m]))
bytes_per_value = sum by (feature)(rate(io_bytes_value_total[5m])) / sum by (feature)(rate(value_events_total[5m]))
mwh_per_1k    = 1000 * power_model_watts * sum(rate(cpu_seconds_value_total[5m])) / sum(rate(value_events_total[5m]))
```

**4) PR template block**

```md
### Efficiency impact (user terms)
- RPV: cpu/value Δ: -12%, bytes/value Δ: -8%
- TTV p95: 720 ms (budget ≤ 1,000 ms) ✅
- mWh/1k actions: -15%
- Estimated € per order: €0.0009 (↓ from €0.0011)
```

### Dashboards
- **CPU/bytes/mWh/€ per value** by feature/tenant/region; **TTV p95**; conversion vs RPV cohorts.
- Top regressions by PR/release; exemplar links to traces.

### CI gates
- Fail if **RPV** (cpu/bytes/value) ↑ >10% or **TTV p95 > 1 s** for labeled actions.
- Require a Value Event for new features; block if absent.