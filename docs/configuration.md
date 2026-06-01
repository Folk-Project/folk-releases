# Configuration

Folk is configured via `folk.toml` in your project root. All settings have sensible defaults — you only need to specify what you want to change.

Environment variables with the `FOLK_` prefix override file settings:

```bash
FOLK_WORKERS_COUNT=8
FOLK_HTTP_LISTEN=0.0.0.0:9000
FOLK_LOG_FILTER=debug
```

## Server

```toml
[server]
rpc_socket = "/tmp/folk.sock"     # Admin RPC Unix socket path
shutdown_timeout = "30s"          # Graceful shutdown timeout
```

## Workers

```toml
[workers]
script = "vendor/bin/folk-worker"  # PHP worker script
php = "php"                        # PHP binary path
count = 4                          # Worker threads (>1 requires ZTS)
max_jobs = 1000                    # Recycle after N requests (0 = never)
ttl = "3600s"                      # Recycle after this lifetime
exec_timeout = "30s"               # Per-request timeout
boot_timeout = "30s"               # Worker boot timeout
warmup = true                      # Opcache warmup before worker spawn
```

!!! note "Opcache warmup"
    When `warmup` is enabled (default), Folk automatically compiles all files from `vendor/composer/autoload_classmap.php` into shared opcache before spawning workers. This eliminates the parse+compile overhead on first requests — workers start with hot opcache immediately. Works with any framework or vanilla PHP project that uses Composer. Requires `composer install --optimize-autoloader` for full coverage.

!!! note "Worker recycling"
    Workers are recycled (terminated and respawned) when they exceed `max_jobs` or `ttl`. This prevents memory leaks from accumulating. The main thread worker is never recycled.

## Logging

```toml
[log]
filter = "info"   # "debug", "info", "warn", "error"
format = "text"   # "text", "json", or "pretty"

[log.plugins]     # Per-plugin log level overrides
http = "warn"     # Only warnings and errors from HTTP plugin
jobs = "debug"    # Verbose logging from jobs plugin
core = "info"     # Core server logging
```

### Per-Plugin Log Levels

The `[log.plugins]` section lets you set per-plugin log levels using friendly names. No need to know Rust crate names — Folk maps them automatically:

| Config key | Rust target |
|------------|-------------|
| `http` | `folk_plugin_http` |
| `jobs` | `folk_plugin_jobs` |
| `grpc` | `folk_plugin_grpc` |
| `metrics` | `folk_plugin_metrics` |
| `process` | `folk_plugin_process` |
| `core` | `folk_core` |
| `ext` | `folk_ext` |

All output goes to stdout. The three formats share the same data structure (timestamp, level, target, message, context fields) — only the visual representation differs.

**text** — compact, one line per event:
```
2026-05-20T14:30:00Z INFO  [http] 200 GET /api/users 12ms
2026-05-20T14:30:01Z WARN  [process] restarting name=scheduler restarts=2
```

**json** — structured, for ELK/Loki/Grafana:
```json
{"ts":"2026-05-20T14:30:00Z","level":"INFO","plugin":"http","msg":"200 GET /api/users","status":200,"duration_ms":12}
```

**pretty** — multi-line, for debugging:
```
  2026-05-20T14:30:00Z INFO [http]
    200 GET /api/users
    status: 200
    duration_ms: 12
```

!!! note
    The `RUST_LOG` environment variable takes precedence over `filter` if set.

## Plugins

Each plugin has its own configuration section. See the plugin pages for details:

- [HTTP](plugins/http.md) — `[http]`
- [Jobs](plugins/jobs.md) — `[jobs]`
- [gRPC](plugins/grpc.md) — `[grpc]`
- [Metrics](plugins/metrics.md) — `[metrics]`
- [Process](plugins/process.md) — `[process]`

## Duration Format

Duration fields accept human-readable values:

| Format | Meaning |
|--------|---------|
| `30s` | 30 seconds |
| `5m` | 5 minutes |
| `1h` | 1 hour |
| `1d` | 1 day |

## Complete Example

```toml
[server]
shutdown_timeout = "10s"

[workers]
script = "vendor/bin/folk-worker"
count = 4
max_jobs = 1000

[log]
filter = "info"
format = "json"

[log.plugins]
http = "warn"
process = "debug"

[http]
listen = "0.0.0.0:8080"
access_log = true

[jobs]
driver = "redis"
redis_url = "redis://127.0.0.1:6379"

[[jobs.queues]]
name = "default"
concurrency = 4
max_retries = 3

[grpc]
listen = "0.0.0.0:50051"
proto = ["proto/service.proto"]
max_recv_message_size = "4mb"
timeout = "30s"

[metrics]
listen = "0.0.0.0:9090"
prefix = "folk"
ready_path = "/ready"

[[metrics.collectors]]
name = "app_requests_total"
type = "counter"
help = "Total application requests"
labels = ["method", "endpoint"]

[[process.processes]]
name = "scheduler"
command = "php artisan schedule:work"
restart = "always"
```

See the [Configuration Reference](reference.md) for a fully commented `folk.toml` with all options.
