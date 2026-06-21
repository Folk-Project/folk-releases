# Configuration Reference

Complete `folk.toml` with all available options and their defaults.

```toml
# =============================================================================
# Folk Application Server — Full Configuration Reference
# =============================================================================
#
# Place as folk.toml in your project root.
# All values below are defaults unless noted otherwise.
#
# Environment variable overrides: FOLK_{SECTION}__{FIELD}  (double underscore)
# Example: FOLK_WORKERS__COUNT=8, FOLK_HTTP__LISTEN=0.0.0.0:9000
# Note: double underscore separates the section from the field name.
#
# Duration format: "30s", "5m", "1h", "1d"
# Size format: "10mb", "512kb", "4096" (bytes)

# =============================================================================
# Server
# =============================================================================
[server]
rpc_socket = "/tmp/folk.sock"          # Admin RPC Unix socket path
shutdown_timeout = "30s"               # Max time for graceful shutdown after SIGTERM

# =============================================================================
# Workers
# =============================================================================
[workers]
script = "vendor/bin/folk-worker"      # PHP worker entry point
php = "php"                            # PHP binary path
count = 4                              # Worker threads (>1 requires PHP ZTS)
max_concurrent_per_worker = 1          # Concurrent requests per worker (only 1 supported; >1 reserved)
max_jobs = 1000                        # Recycle worker after N requests (0 = never)
ttl = "3600s"                          # Recycle worker after this lifetime
exec_timeout = "30s"                   # Per-request execution timeout
boot_timeout = "30s"                   # Max time to wait for worker ready signal
warmup = true                          # Compile Composer classmap into opcache before spawn

# =============================================================================
# Dev mode (hot reload)
# =============================================================================
# Watch PHP files and recycle workers on change — for development only.
# Disabled by default. Requires PHP ZTS with workers.count > 1: the main PHP
# thread (worker #1) is not recyclable, so single-worker servers cannot fully
# hot reload. On change, Folk gracefully recycles recyclable workers after their
# current request completes; each restarted worker re-bootstraps and re-reads
# the changed code. Keep opcache.validate_timestamps = 1 in dev (the default)
# so OPcache picks up edits; do not pair enable_cli = 1 with validate_timestamps = 0.
[dev]
watch = false                          # Enable file watcher + hot reload
watch_paths = ["app", "src", "routes", "config"]  # Directories watched recursively
watch_extensions = ["php"]             # Extensions that trigger a reload
debounce = "300ms"                     # Collapse a burst of file events into one reload

# =============================================================================
# Logging
# =============================================================================
[log]
filter = "info"                        # Log level: "trace", "debug", "info", "warn", "error"
format = "text"                        # Output format: "text", "json", "pretty"

# Per-plugin log level overrides.
# Keys: http, jobs, grpc, metrics, process, core, ext
[log.plugins]
# http = "warn"
# jobs = "debug"
# core = "info"

# =============================================================================
# HTTP Plugin
# =============================================================================
[http]
listen = "0.0.0.0:8080"               # Listening address
read_timeout = "10s"                   # Max time to read request body
write_timeout = "30s"                  # Max time to write response (504 on timeout)
max_request_size = "10mb"              # Max request body ("10mb", "512kb", or bytes)
access_log = false                     # Log every request (method, URI, status, duration)
trusted_proxies = []                   # CIDR subnets for X-Forwarded-For extraction
h2c = false                            # HTTP/2 cleartext (without TLS)

# TLS — enables HTTPS with automatic HTTP/2 via ALPN
# [http.tls]
# cert = "/path/to/cert.pem"
# key = "/path/to/key.pem"

# Response compression
# [http.compression]
# enabled = true
# algorithms = ["gzip", "br", "zstd"]  # Priority order. Also: "deflate"
# min_size = 256                        # Min response size to compress (bytes)

# Lua hook pipeline — zero or more entries
# [[http.hooks]]
# event = "request.before"             # "request.before" | "request.error" | "response.headers" | "response.after"
# lua = "hooks/rate_limit.lua"         # Path to Lua script (relative to working directory)
# mode = "sync"                        # "sync" (critical path) | "async" (fire-and-forget)
# timeout_ms = 5                       # Sync-only: abort hook after N ms (fail_open)
# on_error = "fail_open"               # "fail_open" (skip+WARN) | "fail_closed" (→ 500)

# =============================================================================
# Jobs Plugin
# =============================================================================
[jobs]
driver = "memory"                      # "memory" or "redis"

# Redis connection (when driver = "redis")
host = "127.0.0.1"
port = 6379
password = ""
db = 0

[[jobs.queues]]
name = "default"
concurrency = 4                        # Concurrent consumers for this queue
max_retries = 3                        # Retries before DLQ or discard
retry_delay = "1s"                     # Base delay between retries
retry_backoff = "exponential"          # "exponential", "linear", "fixed"
job_timeout = "60s"                    # Max job execution time ("0s" = unlimited)
# dead_letter_queue = "failed"         # Queue name for failed jobs (omit to discard)
priority = 10                          # Lower number = higher priority

# =============================================================================
# gRPC Plugin
# =============================================================================
[grpc]
listen = "0.0.0.0:50051"              # Listening address
proto = []                             # Proto files for reflection (empty = no reflection)
max_recv_message_size = "4mb"          # Max incoming message size
max_send_message_size = "4mb"          # Max outgoing message size
# timeout = "30s"                      # Server-wide RPC timeout (omit = no timeout)
# max_concurrent_streams = 200         # HTTP/2 concurrent streams limit
compression = false                    # Enable gzip compression

# HTTP/2 keepalive
# [grpc.keepalive]
# interval = "60s"                     # PING interval
# timeout = "20s"                      # PING timeout before disconnect

# TLS — enables secure gRPC
# [grpc.tls]
# cert = "/path/to/cert.pem"
# key = "/path/to/key.pem"

# =============================================================================
# Metrics Plugin
# =============================================================================
[metrics]
listen = "0.0.0.0:9090"               # Listening address
prefix = "folk"                        # Namespace prefix for custom metrics (folk_*)
metrics_path = "/metrics"              # Prometheus scrape endpoint
health_path = "/health"                # Liveness probe (always 200)
ready_path = "/ready"                  # Readiness probe (503 if not ready)

# Per-plugin metric filtering (omit = all enabled)
# [metrics.plugins]
# core = true
# http = true
# jobs = true
# grpc = false                         # Disable folk_grpc_* metrics
# process = true

# Custom metrics — PHP can manipulate via RPC (metrics.increment, metrics.observe, metrics.set)
# [[metrics.collectors]]
# name = "app_requests_total"          # Becomes folk_app_requests_total
# type = "counter"                     # "counter", "gauge", "histogram"
# help = "Total application requests"
# labels = ["method", "endpoint"]

# [[metrics.collectors]]
# name = "request_duration_seconds"
# type = "histogram"
# help = "Request processing duration"
# buckets = [0.01, 0.05, 0.1, 0.5, 1.0, 5.0]

# [[metrics.collectors]]
# name = "queue_depth"
# type = "gauge"
# help = "Current queue depth"

# =============================================================================
# Process Plugin
# =============================================================================
# Managed background processes with supervision and automatic restart.

# [[process.processes]]
# name = "scheduler"
# command = "php artisan schedule:work"
# restart = "always"                   # "always", "on_failure", "never"
# max_restarts = 5                     # Max restart attempts
# restart_delay = "2s"                 # Delay between restarts
# directory = "/app"                   # Working directory (default: server CWD)
# stop_timeout = "5s"                  # Graceful shutdown timeout
# stop_signal = "TERM"                 # Stop signal: TERM, INT, QUIT
# numprocs = 1                         # Process copies (name:0, name:1, ...)
#
# [process.processes.env]
# APP_ENV = "production"
#
# [process.processes.logging]
# stdout = "inherit"                   # "inherit", "null", or { file = "/path" }
# stderr = "inherit"
```
