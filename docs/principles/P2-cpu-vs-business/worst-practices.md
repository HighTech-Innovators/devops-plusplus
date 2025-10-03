---
principle: P2-cpu-vs-business
doc_type: worst-practices
tags: [anti-patterns, waste, misalignment, express, cpu-high]
---

# CPU Load vs. Business Value — Worst Practices (CPU High Focus)

## Avoid this

- **Ignoring CPU plateaus** when dashboards show flat **value_events_total** — runaway loops, hot spins, or noisy exporters.
- **Clock‑driven churn** (scans, heartbeats) that keeps CPU high with no Value Events.
- **Missing correlation dashboards** — you can’t tell if CPU buys value.
- **Synchronous heavy logging/metrics** on hot paths that inflate CPU without increasing value.
- **Unlimited retries** that keep CPU hot on permanent failures.
- **Shared pools kept “warm” 24/7** for rare jobs.

### Express/Node anti‑patterns & fixes

**1) Tight retry loop (bad)**
```js
while(true){ try { await send(); break; } catch(e) {} } // ❌ burns CPU
```
**Fix**
```js
for (let i=0;i<max;i++){ try { await send(); break; } catch(e){ await sleep(backoff(i)); if (isPermanent(e)) break; } }
```

**2) Hot exporter (bad)**
```js
setInterval(() => exporter.flush(), 1000); // ❌ keeps CPU active regardless of demand
```
**Fix**
```js
setInterval(() => exporter.flush(), 10000).unref(); // ✅ longer, unref'd; or event-driven flush
```

**3) “Warmers” pummeling endpoints (bad)**
```js
setInterval(() => ping('/compute-heavy'), 5000);
```
**Fix** — Remove or constrain by KPI; only warm just-in-time.

**4) No value correlation (bad)**
- Alert on CPU alone → **noise**; you miss that no value is produced.

**Fix** — Alert on **CPU high & Value low**; log `cpu_high_value_low` from watchdog.

### Smells checklist
- [ ] CPU>80% for 5+ min while `value_events_total` rate < threshold.
- [ ] No chart showing **V/CPU‑s**.
- [ ] Intervals not `.unref()`‑ed.
- [ ] Retries without backoff or reason labels.
- [ ] Logs/metrics sync on hot paths.