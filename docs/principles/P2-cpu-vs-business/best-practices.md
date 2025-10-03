---
principle: P2-cpu-vs-business
doc_type: best-practices
tags: [value-per-cpu, efficiency, governance, observability, express, cpu-high]
---

# CPU Load vs. Business Value — Best Practices (CPU High Focus)

**Goal:** When CPU is high, it must correlate with **Business Value Events** you can count. If CPU hangs high while value stays flat, treat it as **waste or fault** and react automatically.

## Do this

- **Define Value Events (observable):** Counters you can increment on success, with logs and traces sharing join keys.
  - Examples: `order_committed_total{tenant,region}`, `messages_delivered_total{channel}`.
- **Correlate CPU with Value:** Dashboard & alerts that compare `process_cpu_seconds_total` to `value_events_total`.
  - Track **V/CPU‑s = value_rate / cpu_rate** and alert when it drops below an SLO.
- **Guardrails for high CPU:** Treat “CPU high & Value low” as a **page** (or auto‑mitigation trigger).
- **Bound background work:** Token buckets/backoff to prevent runaway CPU when no value events arrive.
- **Prefer event‑driven:** Compute only on business precursors (webhooks, CDC, queue messages).
- **Idempotency & dedupe:** Prevent double compute when the same value would be produced.
- **Expose reasons:** Label retries with `reason`; label value events with `tenant/feature` for attribution.

## Express/Node — CPU/Value Watchdog (drop‑in)

```js
// cpuValueWatchdog.js
const os = require('os');
const { performance } = require('perf_hooks');

/** Pass in your counter getter for value events and a logger. */
function makeCpuValueWatchdog({ getValueEventsDelta, logger = console, intervalMs = 5000,
  cpuHighPct = 0.80, minValueRate = 0.1 }) {

  let lastCpu = process.cpuUsage();
  let lastTime = process.hrtime.bigint();
  let lastValue = getValueEventsDelta(); // drain on init

  function sample() {
    const now = process.hrtime.bigint();
    const elapsedUs = Number(now - lastTime) / 1000;
    lastTime = now;

    const cpu = process.cpuUsage(lastCpu);
    lastCpu = process.cpuUsage();

    const cpuUserSysUs = cpu.user + cpu.system;          // microseconds on CPU
    const cpuPct = Math.min(1, cpuUserSysUs / (elapsedUs * os.cpus().length)); // crude multi-core estimate

    const valueDelta = getValueEventsDelta();
    const valueRate = valueDelta / (elapsedUs / 1e6);    // events per second

    if (cpuPct >= cpuHighPct && valueRate < minValueRate) {
      logger.warn(JSON.stringify({
        level: 'warn', event: 'cpu_high_value_low',
        cpu_pct: Number(cpuPct.toFixed(3)),
        value_rate_eps: Number(valueRate.toFixed(3)),
        msg: 'CPU high without matching business value'
      }));
    }
  }

  const t = setInterval(sample, intervalMs);
  t.unref(); // do not keep process alive
  return () => clearInterval(t);
}

module.exports = { makeCpuValueWatchdog };
```

**Wire it up in your app:**

```js
// telemetry/valueCounter.js
let count = 0;
module.exports.valueEvent = (name, attrs = {}) => (req, res, next) => {
  res.on('finish', () => { if (res.statusCode >= 200 && res.statusCode < 300) count++; });
  next();
};
module.exports.getValueEventsDelta = () => { const c = count; count = 0; return c; };
```

```js
// app.js
const express = require('express');
const http = require('http');
const { valueEvent, getValueEventsDelta } = require('./telemetry/valueCounter');
const { makeCpuValueWatchdog } = require('./cpuValueWatchdog');

const app = express();
app.use(express.json({ limit: '1mb' }));

// Emit value when orders are successfully committed
app.post('/orders/commit', valueEvent('order_committed'), async (req, res) => {
  // ... persist idempotently
  res.status(201).json({ ok: true });
});

// Start watchdog: page when CPU high but Value flat
const stopWatchdog = makeCpuValueWatchdog({
  getValueEventsDelta,
  logger: console,
  intervalMs: 5000,
  cpuHighPct: 0.80,     // 80% CPU
  minValueRate: 0.1     // < 0.1 successful value events / sec
});

const server = http.createServer(app);
server.keepAliveTimeout = 5000;
server.headersTimeout = 10000;
server.listen(process.env.PORT || 3000);
process.on('SIGTERM', () => { stopWatchdog(); server.close(() => process.exit(0)); });
```

## Observability: queries & alerts (PromQL‑ish)

**CPU high & value low (primary page):**
```
(sum by (service)(rate(process_cpu_seconds_total{service="web"}[5m])) / scalar(count(node_cpu_seconds_total{mode="idle"})))
> 0.8
and sum by (service)(rate(value_events_total{service="web"}[5m])) < 0.1
```

**V/CPU‑s floor (ratio alert):**
```
(sum by (service)(rate(value_events_total{service="web"}[5m])))
/ clamp_min(sum by (service)(rate(process_cpu_seconds_total{service="web"}[5m])), 1e-6)
< 0.02   # tune per service
```

**Background spike (no requests but CPU rising):**
```
sum by (pod)(rate(process_cpu_seconds_total{service="web"}[5m]))
unless sum by (pod)(rate(http_requests_total{service="web"}[5m])) > 0
```

## CI gates (perf job)
- Run a synthetic workload that produces Value Events; assert **V/CPU‑s ≥ baseline**.
- Run a “no‑value” phase (no business events) and verify **CPU duty‑cycle ≤ 5%**.
- Fail PRs where watchdog logs `cpu_high_value_low` under test.

## Acceptance criteria
- Watchdog running in all envs, unref’d.
- Dashboards show CPU vs Value and V/CPU‑s trend.
- Alerts page on CPU high without value; auto‑mitigation documented.