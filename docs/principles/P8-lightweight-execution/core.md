# P8 — Lightweight Execution (Generic Core, with WebAssembly Guidance)

> **Principle:** Prefer small, fast‑starting, low‑footprint execution units that do one job well. Keep hot paths lean; choose runtimes and packaging that minimize **startup time**, **RSS/heap**, and **Ops‑Per‑User‑Action (OPUA)**. Use WebAssembly (Wasm), serverless, micro‑runtimes, or native libs **when they reduce cost/energy for the outcome**.

## Why it matters
- **Latency & UX:** Cold starts and bloated heaps punish tail latency.
- **Cost & Energy:** Smaller binaries and faster start → less idle burn and fewer watts per action.
- **Reliability:** Lean components have fewer moving parts and clearer failure domains.

## What “lightweight” means (signals)
- **Cold start time** (boot → ready): p95 within budget per surface (edge/serverless often ≤ 100–300 ms).
- **RSS/heap at steady state**: target **≤ 60–70%** of limit; trend ↓.
- **OPUA**: cpu_seconds/action trending ↓; avoid heavy frameworks on hot paths.
- **Artifact size**: bundles/images small; no transitive bloat.
- **Duty cycle fit**: idle workloads scale to zero or spawn on demand.

## When to use WebAssembly (decision points)
- The unit of work is **CPU‑bound** (numeric, parsing, crypto, codecs) and benefits from **SIMD/linear memory**.
- You need **portability/sandboxing** across environments (edge/workers/serverless).
- You can keep **marshalling overhead** low (operate on **ArrayBuffer**/shared memory, not JSON copying).
- You can **lazy‑load** the module and meet cold‑start budgets (small `.wasm`, cached instantiation).
- You have a clear **measurement plan** (V/CPU‑s, cold‑start, RSS) to prove the win.

## Observability (what to emit)
- `startup_ms` (cold start), `rss_bytes` steady, `artifact_size_bytes` (from CI).
- `op_cpu_us`/`op_latency_ms` for Wasm vs JS/native path.
- `wasm_instantiates_total`, `wasm_cache_hits_total` (did we reuse instance?).
- Labels: `{surface=edge|serverless|server, impl=wasm|js|native, module=<name>}`.

## PromQL‑ish
```
# Cold start
histogram_quantile(0.95, sum by (le, service)(rate(startup_seconds_bucket[5m])))

# CPU per action
sum by (route)(rate(process_cpu_seconds_total[5m])) / sum by (route)(rate(http_requests_total[5m]))

# Wasm cache effectiveness
sum(rate(wasm_cache_hits_total[5m])) / clamp_min(sum(rate(wasm_instantiates_total[5m])), 1)
```

## CI/CD Guardrails
- ❌ **Cold start p95** regression > **10–20%** vs main → **fail**.
- ❌ **Artifact size** ↑ > **10%** (protected apps) → **fail**.
- ❌ **RSS/OPUA** ↑ > **10%** → **fail**.
- ✅ PR comment: cold‑start, RSS, artifact size, and **Wasm vs JS op latency** deltas.

## Acceptance Criteria
- Lightweight unit per responsibility; measurable cold‑start & memory budgets.
- If Wasm is used: **lazy instantiation**, small module, minimal copies, clear win over JS/native.
- Dashboards show cold start, RSS/heap, OPUA; CI enforces budgets.