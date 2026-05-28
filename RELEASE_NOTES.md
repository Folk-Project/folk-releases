### What's new

- **folk-core v0.2.4 + folk-plugin-http v0.2.2** — structured logging (phase 33):
  - Per-plugin log levels via `[log.plugins]` — friendly names (`http`, `jobs`, `core`) mapped to Rust crate targets
  - JSON format: `flatten_event` for flat structure with `target` field (ELK/Loki/Grafana ready)
  - HTTP access log now includes `response_bytes` field
  - `RUST_LOG` env var still takes precedence over config

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

### Crate versions

| Crate | Version |
|-------|---------|
| folk-plugin-http | 0.2.2 |
| folk-plugin-jobs | 0.3.0 |
| folk-plugin-grpc | 0.2.2 |
| folk-plugin-metrics | 0.2.1 |
| folk-plugin-process | 0.2.1 |
| folk-core | 0.2.4 |
| folk-ext | 0.2.4 |
