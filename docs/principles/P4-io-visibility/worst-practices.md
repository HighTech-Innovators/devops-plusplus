---
principle: P4-io-visibility
doc_type: worst-practices
tags: [anti-patterns, io, latency, retries, buffering, express, node]
---

# I/O Visibility — Worst Practices (Generic + Express)

## Avoid this

- **No timeouts** on HTTP/DB/cache/file operations.
- **Chatty N+1** patterns; loops issuing per‑row upstream calls.
- **Full buffering** of large responses/files (no streaming).
- **Unbounded retries**; ignore reason codes; retry on permanent failures.
- **New TCP/TLS per request**; disabled keep‑alive without justification.
- **No attribution** — can’t tell which dependency eats bytes/latency/errors.
- **Hidden caches** with unknown hit‑rate and eviction.

### Express/Node anti‑patterns & fixes

**1) Missing timeout (bad)**
```js
const res = await fetch(url); // ❌ hangs on slow networks
```
**Fix**
```js
const res = await requestWithTimeout(url, { timeoutMs: 5000, dependency: 'payments' });
```

**2) N+1 in a loop (bad)**
```js
for (const id of ids) {
  await fetch(`https://api/items/${id}`); // ❌ chatty
}
```
**Fix — batch/coalesce**
```js
await fetch(`https://api/items?ids=${ids.join(',')}`);
```

**3) Full buffering (bad)**
```js
const data = await (await fetch(bigUrl)).text(); // ❌
res.send(data);
```
**Fix — stream**
```js
const up = await requestWithTimeout(bigUrl, { dependency: 'large_api' });
up.body.pipe(res);
```

**4) Unbounded retries (bad)**
```js
while(true){ try{ await fetch(url); break; } catch(e){} } // ❌
```
**Fix — backoff+jitter+cap**
```js
for (let i=0;i<3;i++){ try{ await fetch(url); break; } catch(e){ if (isPermanent(e)) break; await sleep(50*Math.pow(2,i)); } }
```

**5) Per‑request connections (bad)**
```js
fetch(url, { headers: { Connection: 'close' } }); // ❌ disables reuse
```
**Fix — keep‑alive + pool**
```js
// node http.Agent with keepAlive:true, and per-host maxSockets cap
```

### Smells checklist
- [ ] p95 I/O latency rising without matching value events.
- [ ] Bytes/req ↑ >10% vs baseline for hot routes.
- [ ] No `io_*` metrics or dependency labels.
- [ ] Multiple similar upstream calls in loops.
- [ ] Large payloads buffered rather than streamed.
- [ ] Retries without reason labels / caps.