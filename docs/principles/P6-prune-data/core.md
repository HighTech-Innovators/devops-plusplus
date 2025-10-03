# P6 — Prune Data (Generic Core)

> **Principle:** Collect, move, and keep only the data that creates user or business value. Reduce volume at the **earliest possible point** (client/edge/source), shape results to what is actually consumed, and expire what is no longer needed.

## Why it matters
- **Cost & Energy:** Less data in motion/storage → fewer bytes, fewer I/O ops, lower watts.
- **Performance:** Smaller payloads and datasets → faster queries and responses.
- **Risk & Governance:** Minimize PII retention windows; reduce breach and compliance surface.

## Core strategies
- **Filter early:** push predicates to producers (edge, CDC filters, DB WHERE), use projections (columns), and drop unused fields.
- **Summarize where sufficient:** aggregates, sketches (HLL, TDigest) for exploration; compute exact only when needed.
- **TTL & retention policies:** automatic expiration with proofs (deletion logs, metrics).
- **Schema evolution:** version fields; deprecate and remove with migration windows.
- **Deduplicate:** idempotency keys; compaction jobs; avoid fan‑out duplication.
- **Selective replication:** replicate only the data needed at each tier/region.
- **Access patterns drive shape:** return only what clients display/use; avoid “kitchen‑sink” responses.

## Signals & Targets
- **Bytes per action** ↓ over time; caps per critical path.
- **Selectivity:** `useful_records / scanned_records` ↑; target > 80%.
- **Field utilization:** % of fields read within N days; remove fields < 5–10% use.
- **TTL conformance:** % of data auto‑expired within window ≥ 99%.
- **Duplication factor:** raw vs unique event ratios trending ↓.

## Observability (PromQL‑ish examples)
```
# Useful vs scanned records
sum(rate(useful_records_total[5m])) / clamp_min(sum(rate(records_scanned_total[5m])), 1)

# Field utilization by name (logs to metrics via processors)

sum by (field)(rate(field_read_total[1d])) / clamp_min(sum by (field)(rate(records_total[1d])), 1)

# Bytes per request
sum by (route)(rate(io_read_bytes_total[5m] + io_write_bytes_total[5m])) / sum by (route)(rate(http_requests_total[5m]))

# TTL success
sum(rate(records_expired_total[5m])) / sum(rate(records_due_to_expire_total[5m]))
```

## CI/CD Guardrails
- ❌ **Bytes/req** regression > 10% on protected routes → **fail**.
- ❌ New fields or endpoints without **utilization plan** (owner, TTL, deprecation path) → **fail**.
- ❌ ETL jobs without **selectivity > X%** or dedupe plan → **fail**.
- ✅ PR auto‑comment: bytes/req delta; fields added/removed; TTL coverage.

## Acceptance Criteria
- Producers apply filters/projections; consumers read shaped results only.
- TTLs configured and monitored; deletion proofs collected.
- Dashboards show bytes/req, selectivity, field utilization, and duplication factor.