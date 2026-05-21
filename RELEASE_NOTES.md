### What's new

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

### Crate versions

| Crate | Version |
|-------|---------|
| folk-plugin-http | 0.2.1 |
| folk-plugin-jobs | 0.3.0 |
| folk-plugin-grpc | 0.2.0 |
| folk-plugin-metrics | 0.2.0 |
| folk-plugin-process | 0.2.0 |
| folk-core | 0.2.3 |
| folk-ext | 0.2.3 |
