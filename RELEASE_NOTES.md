### 0.2.5 — supervisor resilience

**Worker supervisor resilience ([#85](https://github.com/Folk-Project/folk-releases/issues/85)) — `folk-core`/`folk-ext` 0.5.0.**
Hardening the fork-after-warm supervisor with per-worker liveness, observability,
and the recycle policies the fork model was silently skipping.

**Per-worker metrics.** `/metrics` now exposes, labelled by `worker_id`:

- `folk_worker_heartbeat_millis` — wall-clock of each worker's last runtime beat.
- `folk_worker_inflight_seconds` — age of the worker's in-flight request (0 = idle).
- `folk_worker_requests_total` — requests handled per worker slot.

A degraded or hung worker is now visible even without any auto-recycling.

**Liveness watchdog — `[workers] liveness_timeout`** (default `0` = off).
`exec_timeout` only catches a request that runs too long. It can't catch a worker
whose **async runtime** has wedged *outside* a request — e.g. a deadlocked
runtime whose HTTP listener stopped accepting. Each worker's runtime now bumps a
heartbeat every second, independent of PHP and of traffic; if it stalls past
`liveness_timeout` while the process is alive, the master force-recycles it.
Because the heartbeat is traffic-independent, an **idle worker is never mistaken
for a hung one** — set it generously (tens of seconds) or leave it off.

```toml
[workers]
liveness_timeout = "30s"   # force-recycle a worker whose runtime stalls this long
```

**`max_jobs` and `ttl` now work in the fork model (behaviour change).** They
previously had **no effect** under fork-after-warm — the single in-process worker
is the non-recyclable main thread, so the pool skipped them. The master now
enforces them itself (from the per-worker request count and spawn time), so
workers again recycle after `max_jobs` requests or `ttl` age. If you relied on
the (unintended) no-op, set `max_jobs = 0` and a large `ttl` to keep workers
long-lived. `max_memory_mb` (RSS) remains the recommended primary control.

**Recycles are staggered.** When many workers cross a limit at once (memory,
jobs, or ttl), the master recycles **one per second** instead of all together,
avoiding a cold-start stampede where the whole pool respawns and re-warms in
lockstep.

**Streaming timeouts verified.** `exec_timeout` is a dispatch-based deadline
(queue wait isn't counted) and covers streaming responses; a client that
disconnects mid-stream unblocks the PHP writer with an error in bounded time
rather than wedging the worker. No code change — confirmed and documented.

**Scope.** `folk-core`/`folk-ext` only — `folk-api`, the plugins and the PHP
contract are unchanged. Docker-smoked (2 workers): per-worker metrics; a frozen
worker (`kill -STOP`) force-recycled within `liveness_timeout` while an idle
worker is left alone; `max_jobs` recycle + stagger; mid-stream client disconnect;
and HTTP/streaming/recycle regressions.

**Prebuilt extension.** The `0.2.5` prebuilt `folk.so` is rebuilt against
`folk-ext` 0.5.0; no plugin crate changed.
