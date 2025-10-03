---
principle: P8-lightweight-execution
doc_type: worst-practices
tags: [anti-patterns, cold-start, bloat, images, wasm, express, node]
---

# Lightweight Execution — Worst Practices (with WebAssembly pitfalls)

## Avoid this

- **Eager Wasm instantiation at boot** — adds cold‑start latency even if the route is rarely used.
- **Large .wasm modules** — megabyte‑scale modules slow download/compile; split kernels or pre‑compile.
- **Copying buffers repeatedly** between JS↔Wasm — defeats performance; use `ArrayBuffer`/linear memory.
- **Using Wasm for trivial tasks** — overhead > benefit; measure and keep JS for simple ops.
- **Global heavy middleware** — inflates cold path; not lightweight.
- **Always‑on worker pools** — keep idle CPU/mem hot; prefer on‑demand workers/serverless.
- **Bloated images** — shipping dev toolchains/locales into prod.

### Express/Node anti‑patterns & fixes

**1) Eager Wasm (bad)**
```js
// BAD: loads on boot
const buf = fs.readFileSync('./big.wasm');
const inst = await WebAssembly.instantiate(buf);
```
**Fix — lazy + cache**
```js
// instantiate on first use, then reuse
```

**2) Buffer copying (bad)**
```js
const arr = Array.from(req.body); // ❌ copies, boxes
```
**Fix — operate on bytes**
```js
const buf = Buffer.isBuffer(req.body) ? req.body : Buffer.from(req.body || []);
```

**3) Huge module (bad)**
```
// 5MB wasm for a tiny kernel
```
**Fix — slim kernels or native addon for the hot loop; or keep JS if faster end‑to‑end.

**4) Wasm without measurement (bad)**
- No `op_latency`/`startup_ms` metrics; can’t prove benefit.

**Fix — add telemetry** and gate on **Wasm vs JS** latency in CI.

### Smells checklist
- [ ] p95 cold start ↑ vs baseline after adding Wasm.
- [ ] `.wasm` artifact size ↑ >10% with no win in V/CPU‑s.
- [ ] Multiple JS↔Wasm copies per request.
- [ ] Wasm introduced on non‑CPU‑bound paths.
- [ ] Workers kept warm for rare Wasm tasks.