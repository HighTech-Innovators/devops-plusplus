# P2 — CPU Load vs. Business Value (Generic Core, Observability‑First)

> **Principle:** Spend compute only when it advances outcomes you can **see in telemetry**. Define *Business Value* as events you can **count in your observability stack** (metrics/logs/traces), then maximize **Value per CPU‑second (V/CPU‑s)**.

## What counts as Business Value (must be observable)
Pick **1–3 Value Events** per service/job that are emitted as **metrics + logs + traces** with the same keys.
Examples (rename to your domain):
- `order_committed_total{tenant, region}` (counter) and log `level=info event=order_committed order_id=<id> amount=<€>` with trace/span linking `trace_id`.
- `messages_delivered_total{channel}` (counter) + log `event=message_delivered message_id`.
- `etl_records_validated_total{dataset}` (counter) + log `event=etl_record_validated dataset`.

> If it’s not emitted as a metric/log/trace, it does **not** count as Business Value here.

## V/CPU‑s: Value per CPU‑second
Define:
- **Value** = rate of your **Value Events** (counters).
- **CPU** = `process_cpu_seconds_total` (or runtime‑specific CPU metric).
- **V/CPU‑s** = `rate(value_events_total[5m]) / rate(process_cpu_seconds_total[5m])`

Track by meaningful labels (e.g., `tenant`, `region`, `feature_flag`).

## Signals & Targets (observable)
- **V/CPU‑s**: trending **↑**; investigate **drops >10%** w/w.
- **OPUA** (`cpu_seconds / action`): trending **↓**; enforce non‑regression.
- **Waste share** (retry+poll+duplicate CPU): **≤ 10%** of CPU.
- **Precompute utilization**: `value_reads / precompute_outputs` **≥ 80%** within TTL.

## Required instrumentation (copy/paste sketch)
- **Metric:** counter `value_events_total{service,tenant,feature}`.
- **Log:** structured JSON with `event=<value_event>`, `trace_id`, keys that let BI join (e.g., `order_id`, `amount`).
- **Trace:** span named for the value event; attach exemplar linking the counter sample to the trace/span id.

```js
// Pseudocode (Node/OTEL-like)
valueCounter.add(1, { service, tenant, feature });
logger.info({ event: 'order_committed', order_id, amount, trace_id: ctx.traceId });
span.addEvent('order_committed', { order_id, amount });
```

## Decision Guide (telemetry‑anchored)
1) **Can we measure the value?** If not instrumented, instrument **before** scaling or optimizing.
2) **Is compute happening without value signals?** Replace with event‑driven or defer until a value precursor arrives.
3) **Is there duplicate value work?** Use idempotency keys and dedupe in sinks.
4) **Is the value density low?** Abort early; backoff or drop precision.

## CI/CD Guardrails (against telemetry)
- ❌ **V/CPU‑s drop >10%** vs main on perf dataset → **fail** (requires justification link to incident/flag).
- ❌ **OPUA** p95 regression >10% → **fail**.
- ❌ New periodic job without: **Value Event**, **KPI**, **CPU/IO budgets**, **backoff/jitter**, **consumption SLO** → **fail**.
- ✅ PR bot posts: **V/CPU‑s delta**, **mWh/1k actions**, and top labels moving.

## Observability Queries (PromQL‑ish)
```
# Value per CPU‑second
sum by (service)(rate(value_events_total[5m]))
  / sum by (service)(rate(process_cpu_seconds_total[5m]))

# CPU per action (OPUA)
sum by (service)(rate(process_cpu_seconds_total[5m]))
  / sum by (service)(rate(user_actions_total[5m]))

# Waste share
(sum(retry_cpu_seconds[5m]) + sum(polling_cpu_seconds[5m]) + sum(duplicate_cpu_seconds[5m]))
  / sum(process_cpu_seconds_total[5m])
```

## Acceptance Criteria
- Value Events defined and instrumented (metric+log+trace) with stable keys.
- Dashboards show **V/CPU‑s**, **OPUA**, **waste share**, with exemplars to traces.
- Periodic jobs prove **consumption SLOs** (≥80% of outputs read inside TTL) or are pruned.
- CI enforces **V/CPU‑s** and **OPUA** non‑regression.