# P10 — Efficiency In User Terms (Generic Core)

> **Principle:** Express performance and cost in **units users and the business care about** (e.g., orders per minute, mWh per 1k messages, €/feature use), not just CPU%, QPS, or GB. Optimize to **reduce time-to-value and resource-per-value**, and make those trade-offs visible at decision time.

## Why it matters
- **Alignment:** Engineers optimize what they measure; speak the language of *value* and *experience*.
- **Prioritization:** Budgets framed in user terms clarify where to spend compute (and where not to).
- **Sustainability:** mWh/1k actions and CO₂e tie efficiency to environmental goals.

## Core definitions
- **Value Event** — a countable, observable business outcome (e.g., `order_committed`, `message_delivered`).
- **Resource per Value (RPV)** — CPU seconds, bytes, memory, or euros **per Value Event**. Examples:
  - `cpu_seconds_per_order = cpu_seconds_total / order_committed_total`
  - `bytes_per_message = (io_read + io_write) / messages_delivered_total`
  - `mWh_per_1k_actions = power_model(cpu_seconds, mem, watts) / actions × 1000`
- **Time to Value (TTV)** — intent → first feedback → completion (p95).

## Targets (tune per team)
- **RPV trending ↓** quarter over quarter for top flows.
- **TTV p95 ≤ 1 s** for user‑visible actions (see P9).
- **mWh/1k actions** ↓; CO₂e/1k ↓ (region grid factors).
- **Cost per value** (€/order, €/export) within budget; alert on spikes.

## Observability (prom‑ish)
```
# Resource per Value
sum(rate(process_cpu_seconds_total[5m])) / clamp_min(sum(rate(value_events_total[5m])), 1)

# Bytes per Value
sum(rate(io_read_bytes_total[5m]) + rate(io_write_bytes_total[5m])) / clamp_min(sum(rate(value_events_total[5m])), 1)

# Time to Value (server-side observable approximation)
histogram_quantile(0.95, sum by (le, action)(rate(action_duration_seconds_bucket[5m])))
```

## CI/CD Guardrails
- ❌ **RPV (CPU/bytes per value)** regression > **10%** vs main → **fail**.
- ❌ **TTV p95 > 1 s** for user‑visible actions → **fail**.
- ❌ Features without a **Value Event** and cost attribution → **fail**.
- ✅ PR comment: RPV deltas (cpu/bytes/mWh), TTV, and expected €/value change.

## Acceptance Criteria
- Each critical feature defines **Value Events**, **RPV metrics**, and **TTV budgets**.
- Dashboards display **RPV**, **TTV**, **mWh/1k**, and **€/value** per feature/tenant/region.
- Decisions (docs/PRDs) show cost/value impact, not just throughput/latency.