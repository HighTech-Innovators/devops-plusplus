---
principle: P5-validate-performance-early
doc_type: best-practices
tags: [performance, budgets, ci, shift-left, express, node, testing]
---

# Validate Performance Early — Best Practices (Generic + Express)

## Do this

- **Define budgets per critical path** (e.g., `/checkout`, `/search`): p95 latency, OPUA, bytes/req, MPA.
- **Compare against baseline** (`main`) and **fail on deltas** rather than fixed numbers in CI.
- **Automate perf on PRs** with a quick harness; run heavier soak/canary on merge/nightly.
- **Use golden datasets** representative of prod distributions; include hot/cold paths and cache states.
- **Capture traces** in perf runs; attach exemplars from metrics to traces.
- **Enforce I/O hygiene**: timeouts, backoff, single‑flight, and dependency budgets included in tests.
- **Profile early**: CPU/heap profiles for hot routes under load and leak checks with soak.
- **Parameterize environment**: consistent CPU/memory limits, connection pools, and timeouts for repeatability.

### Express/Node examples

**1) Supertest + budget assertions (smoke)**

```js
// test/budget.smoke.test.js
const request = require('supertest');
const app = require('../app'); // your express app

test('GET /hello meets fast budget', async () => {
  const t0 = Date.now();
  const res = await request(app).get('/hello').expect(200);
  const dt = Date.now() - t0;
  expect(dt).toBeLessThan(50); // local dev smoke budget
});
```

**2) Autocannon micro-load with p95 gate (CI)**

```js
// scripts/bench-hello.js
const autocannon = require('autocannon');
(async () => {
  const r = await autocannon({ url: process.env.URL || 'http://localhost:3000/hello', connections: 20, duration: 20 });
  const p95 = r.latency.p95;
  console.log(JSON.stringify({ event: 'bench_done', latency_p95_ms: p95 }));
  const base = Number(process.env.BASELINE_P95_MS || 60); // fetched from main’s artifact ideally
  if (p95 > base * 1.10) { // >10% regression
    console.error('Latency regression:', p95, 'vs', base);
    process.exit(1);
  }
})();
```

**3) Measure OPUA & MPA during PR runs**

```js
// middleware/perf-metrics.js
const onFinished = require('on-finished');
module.exports = (req, res, next) => {
  const cpu0 = process.cpuUsage(); const rss0 = process.memoryUsage().rss;
  onFinished(res, () => {
    const cpu = process.cpuUsage(cpu0);
    const rss1 = process.memoryUsage().rss;
    console.log(JSON.stringify({ event: 'perf_sample', route: req.route?.path, usr_us: cpu.user, sys_us: cpu.system, rss_delta: rss1 - rss0 }));
  });
  next();
};
```

**4) Golden dataset harness**

```js
// scripts/seed.js – seed representative data (sizes, hotness distribution)
```

**5) Leak check (soak)**

```js
// scripts/soak.js
// run traffic for N minutes and assert RSS slope ~ 0 (no steady growth)
```

### Dashboards (perf env)
- p50/p95 latency by route; OPUA and bytes/req trends.
- Heap/RSS with leak slope; I/O bytes/req; error rate with reason codes.
- Top slow traces; flamegraphs from CPU profiles.

### CI suggestions
- Job A (PR): quick autocannon + budget gates; export artifacts with baseline deltas.
- Job B (nightly): soak + profile; publish flamegraphs and leak results.