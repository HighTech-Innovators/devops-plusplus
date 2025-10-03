# P7 — Smart Compute Location (Generic Core)

> **Principle:** Run work **where it makes the most sense** for latency, cost/energy, data gravity, privacy, and resilience. Choose among **client, edge, server, worker, data store, or serverless** — and re-evaluate as usage and constraints change.

## Why it matters
- **Latency & UX:** Move compute closer to the user or data to reduce RTT and tail jitter.
- **Cost & Energy:** Minimize egress and idle spend; match resource class to duty cycle (scale-to-zero where idle).
- **Compliance & Risk:** Keep PII within region; reduce blast radius by isolating heavy jobs from front doors.

## Location options (mental model)
- **Client** (browser/mobile/desktop): great for rendering/validation; poor for secrets; variable performance.
- **Edge** (CDN/PoP/Workers): fast for auth, routing, caching, small transforms; tight CPU/mem/time budgets.
- **Server (front door)**: session, orchestration, light business logic; avoid heavy CPU/IO here.
- **Worker / Batch**: CPU/GPU heavy work, retries, long I/O; can drain queues and run off-peak.
- **Data store / Pushdown**: push filters/aggregations to DB/OLAP/search for data-local compute.
- **Serverless / Evented**: bursty workloads; natural scale-to-zero; watch cold-start paths & limits.

## Decision guide
1. **Data gravity:** Where does the data already live? Push computation there first (DB pushdown, OLAP).
2. **Latency:** Does the user perceive this? If yes, prefer edge/client; if not, move to workers or batch.
3. **Privacy/Compliance:** Keep PII/regulatory data in-region; avoid moving secrets to client/edge.
4. **Duty cycle:** Idle heavy? Use serverless or spin-up workers on demand.
5. **Failure scope:** Keep front doors simple; isolate high-risk compute behind queues.
6. **Egress & Cost:** Avoid cross-region chatter; cache at the edge; compress at the boundary.

## Signals & Targets
- **RTT-sensitive paths** handled at **edge/client**; p95 ↓ vs server-only baseline.
- **Egress bytes** per action trending ↓; region-local hit-rate ↑.
- **Front-door CPU/IO share** ≤ **X%** of total compute for the feature (heavy in workers).
- **Queue SLAs** met; worker duty cycle ≤ **Y%** at idle; scale-to-zero validated.
- **Pushdown ratio**: % of queries using projections/filters/aggregations at source ≥ **80%**.

## Observability (PromQL-ish)
```
# Edge vs server latency comparison
histogram_quantile(0.95, sum by (le, surface)(rate(http_server_duration_seconds_bucket[5m])))

# Egress per action (by region)
sum by (region)(rate(egress_bytes_total[5m])) / sum by (region)(rate(value_events_total[5m]))

# Pushdown usage
sum(rate(db_pushdown_queries_total[5m])) / sum(rate(db_queries_total[5m]))
```

## CI/CD Guardrails
- ❌ New heavy compute in **front-door** without a move-to-worker plan → **fail**.
- ❌ Cross-region calls without caching or region-local alternative → **fail**.
- ❌ Queries without **pushdown** (filters/projections/aggregations) on large datasets → **fail**.
- ✅ PR auto-comment: predicted p95 delta if moved to edge/worker; egress savings estimate.

## Acceptance Criteria
- Location choice justified (latency, cost, compliance), with measurable targets.
- Edge/client/server/worker responsibilities documented; handoff via events/queues.
- Dashboards show latency & egress by surface (client/edge/server/worker) and pushdown usage.