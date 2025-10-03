---
principle: P5-validate-performance-early
doc_type: worst-practices
tags: [anti-patterns, performance, ci, budgets, express]
---

# Validate Performance Early — Worst Practices (Generic + Express)

## Avoid this

- **No perf checks on PRs** — performance only tested after staging/prod.
- **Absolute thresholds only** — brittle numbers that ignore hardware variance; no baseline compare.
- **Micro‑benchmarks without realism** — no network/DB/cache; results don’t predict prod.
- **Ignoring I/O controls** — tests don’t set timeouts/backoff/pools → passing tests that would fail in prod.
- **Not measuring OPUA/bytes/req/MPA** — only latency checked; regressions slip through.
- **One‑off manual runs** — no artifacts, no dashboards, no deltas.
- **No leak/soak tests** — memory creep and queue growth discovered in incidents.

### Express/Node anti‑patterns & fixes

**1) “It’s fast on my laptop” (bad)**
```sh
node app.js; ab -n 10 http://localhost:3000/hello  # meaningless
```
**Fix** — run autocannon with CI‑consistent params and compare to baseline.

**2) Latency‑only testing (bad)**
- Ignores OPUA/bytes/req/MPA; CPU/memory bloat sneaks in.

**Fix** — capture CPU usage deltas and RSS per request in middleware and assert non‑regression.

**3) No golden data (bad)**
- Tests use trivial payloads; caches are empty or unrealistically hot.

**Fix** — seed data with realistic distributions; test both cold and warm paths.

**4) No traces/profiles (bad)**
- When perf regresses, there’s no flamegraph/traces to explain why.

**Fix** — capture CPU profiles & traces during CI; upload as artifacts.

**5) Hard thresholds on noisy CI (bad)**
- Fixed p95=80ms; CI variance causes flakes.

**Fix** — compare **relative** to main’s artifact (±10%) and require multiple runs or medians.