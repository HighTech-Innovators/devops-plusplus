---
principle: P8-lightweight-execution
doc_type: best-practices
tags: [lightweight, cold-start, wasm, serverless, images, observability, express, node]
---

# Lightweight Execution — Best Practices (with WebAssembly examples)

## Do this

- **Set budgets**: cold‑start p95, RSS steady, artifact size, OPUA per critical path.
- **Pre‑bundle & tree‑shake**: esbuild/rollup; ESM; remove dead code and locales.
- **Lazy‑load heavy code**: dynamic imports; spawn workers on demand.
- **Use Wasm where it wins**: SIMD/CPU‑bound kernels; sandboxed/portable; prove it via **V/CPU‑s** and op latency.
- **Minimize marshaling**: operate on `ArrayBuffer`/`Uint8Array` directly; avoid JSON encode/decode between JS↔Wasm.
- **Stream I/O** & backpressure; push compression/crypto to edges or Wasm/native helpers off the hot path.
- **Tiny images**: multi‑stage builds; distroless/slim; no dev toolchains in runtime images.
- **Right‑size resources**: autoscale on value‑linked signals; scale‑to‑zero for bursty tasks.
- **Telemetry minimal** on hot path; exemplars to traces.

## Express/Node — Wasm patterns (copy/paste)

**1) Lazy, cached instantiation**

```js
// wasm/loader.mjs
import { readFile } from 'node:fs/promises';
let _inst; export async function getWasm() {
  if (_inst) return _inst;
  const buf = await readFile(new URL('./sum.wasm', import.meta.url));
  _inst = (await WebAssembly.instantiate(buf, {})).instance;
  return _inst;
}
```

**2) Zero‑copy(ish) buffer handoff**

```js
// wasm/use.mjs
export function writeTo(memory, buf, offset = 0) {
  const bytes = new Uint8Array(memory.buffer);
  if (offset + buf.length > bytes.length) memory.grow(Math.ceil((offset + buf.length - bytes.length)/65536));
  bytes.set(buf, offset);
}
```

**3) Express route using Wasm, with JS fallback + timing**

```js
import express from 'express';
import { getWasm } from './wasm/loader.mjs';
import { writeTo } from './wasm/use.mjs';

const router = express.Router();
router.post('/wasm/sum', express.raw({ type: '*/*', limit: '1mb' }), async (req, res) => {
  const buf = Buffer.isBuffer(req.body) ? req.body : Buffer.from(req.body || []);
  const jsStart = process.hrtime.bigint(); let s=0; for (let i=0;i<buf.length;i++) s+=buf[i];
  const jsMs = Number(process.hrtime.bigint()-jsStart)/1e6;

  const inst = await getWasm(); const { memory, sum } = inst.exports;
  writeTo(memory, buf, 0);
  const wStart = process.hrtime.bigint(); const wSum = sum(0, buf.length);
  const wMs = Number(process.hrtime.bigint()-wStart)/1e6;

  console.log(JSON.stringify({ event:'op_latency', impl:'js', ms:jsMs }));
  console.log(JSON.stringify({ event:'op_latency', impl:'wasm', ms:wMs }));
  res.json({ ok:true, size:buf.length, js:{sum:s,ms:jsMs}, wasm:{sum:wSum,ms:wMs} });
});
export default router;
```

**4) Cold‑start & artifact size in CI**

```bash
# Build & record bundle size
npx esbuild server.js --bundle --platform=node --outfile=out.js
stat -c "%s" out.js > artifact.size

# Cold-start probe (pseudo)
node -e "const t=Date.now(); require('./out.js'); setTimeout(()=>console.log(Date.now()-t),0)"
```

### Dashboards
- Cold start p50/p95 by service/surface; RSS/heap steady; OPUA per route.
- **Wasm vs JS op latency**; `wasm_instantiates_total` vs `wasm_cache_hits_total`.
- Artifact/container size (from CI); worker spawn rates; lazy import misses.

### CI gates
- Fail on cold‑start p95, artifact size, RSS, or OPUA regression >10% vs baseline.
- Require **Wasm vs JS** op latency sample if Wasm is introduced; fail if Wasm is slower without justification.