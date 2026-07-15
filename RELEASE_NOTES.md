### Adapter fix — `folk-server` via the Composer bin-proxy ([#79](https://github.com/Folk-Project/folk-releases/issues/79))

**You can now launch the server with `php vendor/bin/folk-server`.** Previously
only the direct path (`php vendor/folk/<adapter>/bin/folk-server`) worked; the
canonical Composer bin-proxy `vendor/bin/folk-server` crashed at startup because
Composer's proxy sets the autoload path with a literal `..`
(`vendor/bin/../autoload.php`) that string `dirname()` does not collapse — the
project root resolved to `vendor/bin` and bootstrap failed.

The adapter entrypoints now normalize the path with `realpath()`, so both the
bin-proxy and the direct path / symlink work. The docs now use
`vendor/bin/folk-server` for every framework.

Rust unchanged (PHP-only) — no extension rebuild needed. Ships in **folk/laravel
0.4.3**, **folk/spiral·symfony·yii3 0.2.2**, **folk/sdk 0.4.2** (bin/folk-worker
autoload aligned). Validated end-to-end against a real Composer bin-proxy on all
four framework smoke stands, with a negative control confirming the pre-fix crash.

### 0.2.7 — per-plugin worker pools + cross-pool jobs ([#81](https://github.com/Folk-Project/folk-releases/issues/81))

**Scale subsystems independently.** You can now set a worker count per plugin
instead of one global `[workers] count`:

```toml
[workers]
count = 4        # shared pool: plugins without their own `workers`

[http]
workers = 8      # dedicated pool of 8 processes for HTTP

[jobs]
workers = 1      # dedicated pool of 1 consumer
```

A plugin with an explicit `workers = M` is carved into its own pool of M
processes; plugins without one share the `[workers] count` pool. With **no**
per-plugin `workers`, behaviour is unchanged — a single shared pool of `count`.

**Cross-pool jobs.** Because Folk's RPC is in-process (zero-IPC), the `jobs`
plugin is booted in *every* worker pool so `jobs.push` resolves when you dispatch
a job from an HTTP handler — but it only **consumes** in its own pool. This is
the classic "many web workers, few queue consumers" layout. Cross-pool delivery
goes through the queue backend, so it requires **redis or a managed broker**;
the `memory`/`embedded` drivers stay per-process (a dedicated `jobs` pool with
them warns that pushed jobs won't reach the consumer pool).

Per-worker metrics (`folk_worker_heartbeat_millis`/`_inflight_seconds`/
`_requests_total`) now carry a `pool` label alongside `worker_id`.

**Versions.** `folk-api` 0.3.2 (additive: `PoolContext` + `runs_in_every_pool`,
no plugin cascade), `folk-core`/`folk-ext` 0.6.0, `folk-plugin-jobs` 0.7.0,
`folk-builder` 0.2.16. `folk-plugin-http`/`grpc`/`metrics`/`process`, `folk/sdk`
and the PHP adapters are unchanged. Prebuilt `.so` rebuilt (build-manifest 0.2.7).
