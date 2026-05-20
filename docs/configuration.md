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
max_memory_mb = 256                # Recycle if RSS exceeds this
exec_timeout = "30s"               # Per-request timeout
boot_timeout = "30s"               # Worker boot timeout
```

!!! note "Worker recycling"
    Workers are recycled (terminated and respawned) when they exceed `max_jobs`, `ttl`, or `max_memory_mb`. This prevents memory leaks from accumulating. The main thread worker is never recycled.

## Logging

```toml
[log]
filter = "info"   # "debug", "info", "warn", "error", or "folk_core=trace"
format = "text"   # "text", "json", or "pretty"
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

[http]
listen = "0.0.0.0:8080"

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

[metrics]
listen = "0.0.0.0:9090"

[[process.processes]]
name = "scheduler"
command = "php artisan schedule:work"
restart = "always"
```

See [`folk.example.toml`](https://github.com/Folk-Project/folk-releases/blob/main/folk.example.toml) for a fully commented reference.
