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
