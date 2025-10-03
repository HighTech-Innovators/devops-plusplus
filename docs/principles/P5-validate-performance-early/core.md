# P5 — Validate Performance Early (Generic Core)

> **Principle:** Catch performance problems at the **PR stage**, not after they ship. Treat latency, throughput, CPU, memory, and I/O as **first‑class acceptance criteria** with fast, automated checks and realistic datasets.

## Why it matters
- **Cost & energy:** Fixing OPUA/bytes/req regressions early prevents wasteful runtime spend.
- **Reliability:** Early load/soak tests surface contention, lockups, and resource leaks before prod.
- **Velocity:** Performance budgets embedded in CI avoid late fire drills.

## Core ideas
- **Shift‑left**: run lightweight perf checks on every PR; heavier suites on merge/nightly.
- **Relative gates**: compare against `main`/baseline to avoid brittle absolute thresholds.
- **Representativeness**: use **golden datasets** and traffic shapes that match production.
- **Multi‑signal**: validate **latency (p50/p95)**, **OPUA (cpu_seconds/action)**, **bytes/req**, **memory per action**, **error rate**, **tail jitter**.
- **Budget ownership**: every team owns explicit budgets and SLOs per critical path.

## Signals & Targets (tune per team)
- **Latency**: p95 within SLO per route/job; jitter bounded.
- **OPUA**: non‑regression vs baseline; trend ↓.
- **Bytes/req**: non‑regression; trend ↓.
- **Memory per action (MPA)**: non‑regression; RSS/heap slope ≈ 0 under soak.
- **Error rate**: < 0.5–1% in test; reason codes visible.

## Observability (perf env)
Ensure the perf environment exports the same metrics as prod so dashboards/alerts work on day one.

## CI/CD Guardrails
- ❌ p95 **latency** regression > **10%** on protected routes → **fail**.
- ❌ **OPUA** regression > **10%** → **fail**.
- ❌ **Bytes/req** or **MPA** regression > **10%** → **fail**.
- ❌ Missing perf artifacts (scripts, datasets) → **fail**.
- ✅ Post PR comment: p95, OPUA, bytes/req, MPA deltas; link to traces.

## Acceptance Criteria
- Fast perf test harness runs on PRs (<5–8 minutes).
- Golden dataset + traffic profile checked in (or reproducible).
- Dashboards for perf env; SLOs mirrored from prod.
- Perf budgets documented per critical path and enforced in CI.