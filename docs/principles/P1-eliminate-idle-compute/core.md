# P1 — Eliminate Idle Compute (Generic Core)

> **Principle:** Tie compute to real demand. When there’s nothing to do, the system should approach **zero work** and, when possible, **not be running at all**.

## What “Idle” Means (generic)
**Idle** is a state where a service, job, or function consumes near‑zero CPU and minimal memory/network when no useful work is present. Prefer **event‑driven triggers** over polling, minimize background activity, and allow components to **shut down** after sustained inactivity (scale to zero). Ensure telemetry, logging, and keep‑alive/config choices do not prevent deep idleness.

## Why This Matters
- **Energy & Cost:** Unnecessary uOps burn power and money without user value.
- **Performance:** Idle churn increases contention/GC and deteriorates tail latency.
- **Sustainability:** Less idle compute reduces carbon footprint across the fleet.

## DevOps++ Energy Model
- **Formula:** `mWh = (uOps ÷ GHz) × power_draw`  
- Optimize **Ops‑Per‑User‑Action (OPUA)** first; scale out last.
- Track **background duty‑cycle** (CPU% consumed with no useful work).

## Key Signals & Targets (set per team)
- **OPUA (ops/user action)** — proxy via `cpu_seconds / request` (for services) or `cpu_seconds / output_unit` (for batch).  
  _Target:_ steady ↓ over time; enforce non‑regression.
- **Background CPU duty‑cycle** during zero‑load windows.  
  _Target:_ ≤ **5%** (tighten as you mature).
- **I/O amplification** — extra reads/writes per unit of value.  
  _Target:_ batch/coalesce to reduce.
- **Keep‑alive & wakeups** — timer/connection events when quiescent.  
  _Target:_ near‑zero wakeups at idle.
- **Memory footprint** — working‑set stability when idle; no growth/GC storms.  
  _Target:_ stable and small.
- **Startup cold‑path** — time/cpu to first useful work.  
  _Target:_ fast boot to enable scale‑to‑zero patterns.

## Design Patterns (Do This)
- **Event‑driven > polling:** prefer queues, webhooks, file watchers, pub/sub, or change data capture.
- **Backoff + jitter for any periodic work;** cap duty‑cycle and allow skipping when no work.
- **Offload CPU‑heavy tasks** to worker tiers or managed services; keep front‑door surfaces lean.
- **Lazy initialization:** load heavy modules/data only when needed; defer caches until first use.
- **Reasonable connection/timeouts:** avoid long‑lived zombies (HTTP/TCP/DB). Tune per client and infra.
- **Sampled, asynchronous telemetry:** measure what matters without keeping hot paths busy.
- **Scale‑to‑zero aware design:** small memory footprint and fast start/stop to enable on‑demand runs.
- **Data minimization:** prune at source; avoid precomputing work no one consumes.

## Anti‑Patterns (Don’t Do This)
- **Busy polling** resources for status; periodic “warmers” without a KPI.
- **Global heavyweight middleware** on every request or message.
- **Synchronous, chatty logging** in hot paths; blocking exporters.
- **Permanent worker pools** for rare jobs; always‑warm executors without demand.
- **No shutdown path:** processes linger due to connections/timers; platforms can’t scale down.
- **Unbounded parsing/buffering** causing CPU/GC churn after idle periods.
- **Health checks that trigger heavy work** instead of O(1) probes.

## CI/CD Guardrails (copy/paste & tune)
- ❌ **OPUA non‑regression:** p95 `cpu_seconds / action` ↑ > **10%** vs main on same dataset → **fail**.
- ❌ **Idle duty‑cycle cap:** background CPU > **5%** for 5 consecutive minutes in perf env → **fail**.
- ❌ Any new periodic/daemon task must declare **trigger, expected rate, CPU budget, backoff & jitter**; missing → **fail**.
- ❌ **Timeout hygiene:** service lacks explicit connections/timeouts for its main protocols → **fail**.
- ✅ Auto‑comment on PR: projected **mWh per 1k actions** (DevOps++ model) with before/after deltas.

## Runtime Controls (choose per stack)
- **Network:** set keep‑alive and header/idle timeouts; drop idle connections; enable server‑side backpressure.
- **Workers/Executors:** dynamic pools; spawn on‑demand; terminate after quiet period.
- **Schedulers:** token‑bucket or cron with skip windows; avoid fixed short intervals.
- **Caches:** single‑flight for stampede protection; TTLs based on producer freshness.
- **Parsing/Serialization:** bound payload sizes; prefer streaming for large objects.
- **Resource Hints:** CPU/memory limits sized to steady‑state; enable autoscaling on real signals (QPS, lag).

## Observability Hints (generic)
Use your APM/metrics stack; below are neutral patterns:

**Per‑action CPU (services):**
```
sum(rate(process_cpu_seconds_total[5m])) by (endpoint)
  / sum(rate(actions_total[5m])) by (endpoint)
```
**Per‑unit CPU (batch/pipeline):**
```
sum(rate(process_cpu_seconds_total[5m])) / sum(rate(output_units_total[5m]))
```
**Background duty‑cycle window (no demand):**
```
sum(rate(process_cpu_seconds_total[5m]))
  unless sum(rate(demand_signals_total[5m])) > 0
```
**Wakeups proxy (timers/event‑loop):**
```
rate(runtime_eventloop_wakeups_total[5m])
```

> Replace metric names with your stack’s equivalents (OpenTelemetry, Prometheus, CloudWatch, etc.).

## Acceptance Criteria (Definition of Done)
- In a representative **idle window** (e.g., 10 minutes):
  - Background CPU duty‑cycle **≤ 5%**; no busy loops; no unbounded timers.
  - Memory footprint stable; no GC storms or RSS creep.
  - Component **stops or sleeps** per policy (eligible for scale‑to‑zero).
- **OPUA** gate passes (no regression).
- Telemetry/logging **sampled** and non‑blocking; heavy work offloaded.
- All periodic tasks declare **KPI, trigger, budget, backoff/jitter** and are observable.

## Example Directories
Concrete samples live under `examples/<stack>/` (e.g., `express/`, `django/`, `go-chi/`, `serverless/`).  
Each example must include:
- `meta.yaml` (owner, maturity, targets: OPUA, background duty‑cycle)
- a runnable example
- `best-practices.md` and `worst-practices.md`

---

**DevOps++ Checklist (Ship‑it)**
- [ ] Demand‑driven (events) over polling; backoff+jitter where polling is unavoidable.
- [ ] Background duty‑cycle capped and measured; schedulers budgeted.
- [ ] Offload heavy compute; small boot path; supports scale‑to‑zero.
- [ ] Timeouts/keep‑alives tuned; no zombie connections.
- [ ] OPUA & idle gates enforced in CI; dashboards provide per‑action visibility.