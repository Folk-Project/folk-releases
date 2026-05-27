### What's new

- **Process plugin v0.2.1** — full config expansion (phase 32):
  - Environment variables per process (`[process.processes.env]`)
  - Working directory (`directory = "/app"`)
  - Configurable stop timeout (`stop_timeout`, was hardcoded 5s)
  - Stop signal selection: `TERM`, `INT`, `QUIT`
  - Multiple process instances (`numprocs = 4` → name:0..name:3)
  - stdout/stderr capture: `inherit`, `null`, or `file` (append mode)
  - Shell-aware command parsing via shell-words (quoted arguments)
  - `process.restart` RPC — restart a named process on demand
  - 5 Prometheus metrics: `folk_process_up`, `folk_process_restarts_total`, `folk_process_uptime_seconds`, `folk_process_exit_code`, `folk_process_status`

- **Metrics plugin v0.2.1** — full config expansion (phase 31):
  - `/ready` readiness probe (Kubernetes), `/health` simplified to liveness
  - Configurable endpoint paths (`metrics_path`, `health_path`, `ready_path`)
  - Metric prefix (`folk_*`)
  - Per-plugin metric filtering via `[metrics.plugins]` (core/http/jobs/grpc/process)
  - Declarative custom metrics from config (`[[metrics.collectors]]` — counter, gauge, histogram)
  - RPC: `metrics.increment`, `metrics.observe`, `metrics.set`, `metrics.render`

- **gRPC plugin v0.2.2** — production config expansion (phase 30):
  - TLS/SSL via rustls (`[grpc.tls]`)
  - HTTP/2 keepalive (`[grpc.keepalive]`)
  - Max message size limits (`max_recv_message_size`, `max_send_message_size`)
  - Server-wide RPC timeout
  - Max concurrent streams
  - gRPC compression (gzip)
  - gRPC Health Checking Protocol (grpc.health.v1) — always enabled
  - 6 Prometheus metrics (`folk_grpc_*`)

- **Jobs plugin v0.3.0** — production-ready queue configuration:
  - Configurable retry: `retry_delay` (e.g. `"1s"`) and `retry_backoff` (`exponential`, `linear`, `fixed`)
  - Job execution timeout via `job_timeout` (e.g. `"60s"`, `"0s"` = no limit)
  - Dead letter queue: `dead_letter_queue = "failed"` — failed jobs go to DLQ instead of being discarded
  - Delayed jobs: `delay` field in `jobs.push` RPC for scheduling future execution
  - Priority queues: `priority` field per queue (lower = higher priority)
  - Unified connection config: `host`/`port`/`password`/`db` instead of `redis_url`
  - Redis connection reuse (multiplexed connection stored, not reconnected per operation)
  - JSON-only RPC — msgpack dependency removed entirely (no more PHP ext-msgpack needed)
  - Zero-copy worker dispatch via `execute_value` path
  - Metrics: `folk_jobs_pushed_total`, `folk_jobs_processed_total`, `folk_jobs_processing_duration_seconds`, `folk_jobs_retries_total`, `folk_jobs_dead_letter_total`, `folk_jobs_active`

- **folk/laravel v0.3.0** — companion PHP update:
  - `FolkQueue` uses `json_encode` instead of `msgpack_pack` (no ext-msgpack dependency)
  - `FolkJobHandler` supports new payload wrapper format from jobs plugin v0.3.0

### Breaking changes

- Jobs plugin config: `redis_url` replaced by `host`/`port`/`password`/`db`
- Jobs RPC: `jobs.push` now expects JSON payload (msgpack no longer accepted)
- Requires `folk/laravel` >= v0.3.0 for job dispatch compatibility
- Metrics plugin: `/health` is now a simple liveness probe (always 200). Use `/ready` for readiness checks.

### Crate versions

| Crate | Version |
|-------|---------|
| folk-plugin-http | 0.2.1 |
| folk-plugin-jobs | 0.3.0 |
| folk-plugin-grpc | 0.2.2 |
| folk-plugin-metrics | 0.2.1 |
| folk-plugin-process | 0.2.1 |
| folk-core | 0.2.3 |
| folk-ext | 0.2.3 |
