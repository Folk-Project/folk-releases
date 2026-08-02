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
# A plugin is loaded only when its config section is present. Omit a section to
# disable that plugin (it won't start or log); an empty section enables it with
# defaults. There is no "enabled = false" flag.
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
shutdown_timeout = "30s"               # Max time for graceful shutdown after SIGTERM

# =============================================================================
# on_init — pre-start lifecycle hooks
# =============================================================================
# Commands Folk runs itself BEFORE the framework boots and any listener binds,
# so "everything in one container" needs no separate entrypoint.sh. Runs at the
# top of the entry script, before vendor/autoload.php is required. Absent section
# or empty `commands` = no-op (startup unchanged). Needs a shell (sh -c); cannot
# bootstrap a completely empty vendor/ (the launcher lives in vendor/bin).
[on_init]
exec_timeout = "60s"                   # Default per-step timeout
exit_on_error = true                   # Default: non-zero exit / timeout ABORTS startup (fail-fast;
                                       # differs from RoadRunner's log-and-continue). Opt out per step.
commands = [                           # Run sequentially, in order, cwd = project root, via `sh -c`
  "cp -n .env.example .env",           #   bare string uses the section defaults
  { run = "php artisan migrate --force", exec_timeout = "300s" },  # inline table overrides fields:
                                       #   run / exec_timeout / env / user / exit_on_error
]

# =============================================================================
# Workers
# =============================================================================
[workers]
script = "vendor/bin/folk-worker"      # PHP worker entry point
php = "php"                            # PHP binary path
count = 4                              # Size of the SHARED worker pool: processes carrying every
                                       # plugin that has no per-plugin `workers` (see below)
max_jobs = 1000                        # Recycle worker after N requests (0 = never)
ttl = "3600s"                          # Recycle worker after this lifetime
exec_timeout = "30s"                   # Per-request HARD deadline: a watchdog kills the worker
                                       # process on overrun and the master respawns it
max_memory_mb = 256                    # Recycle a worker whose RSS exceeds this (example; disabled by default)
boot_timeout = "30s"                   # Max time to wait for worker ready signal
destroy_timeout = "10s"                # Grace after recycle SIGTERM before the master SIGKILLs
                                       # the worker's process group (wedged worker won't hang the pool)
liveness_timeout = "0s"                # Force-recycle a worker whose runtime heartbeat stalls this
                                       # long (wedged async runtime); 0 = disabled (metrics-only)
warmup = true                          # Compile Composer classmap into opcache before spawn

# -----------------------------------------------------------------------------
# Per-plugin worker pools
# -----------------------------------------------------------------------------
# Scale subsystems independently by adding `workers = N` to a plugin's section
# (`[http]`, `[jobs]`, `[grpc]`). That plugin gets its OWN dedicated pool of N
# processes carrying only it; plugins without a `workers` key share the
# `[workers] count` pool above. With no per-plugin `workers` anywhere, there is a
# single shared pool of `count` — identical to earlier releases (zero change).
#
#   [http]
#   workers = 8      # 8 HTTP-only processes
#   [jobs]
#   workers = 1      # 1 queue consumer, independent of HTTP concurrency
#
# Cross-pool jobs: dispatching a job from an HTTP handler works across pools, but
# only through the queue backend — use redis or a managed broker. The `memory`
# and `embedded` drivers are per-process, so a dedicated `jobs` pool with them
# cannot receive jobs pushed from other pools (a startup warning is logged).
# Per-worker metrics carry a `pool` label alongside `worker_id`.

# =============================================================================
# Dev mode (hot reload)
# =============================================================================
# Watch PHP files and reload on change — for development only. Disabled by
# default. On change, the master drains its workers and re-execs itself for a
# clean bootstrap + fork, so every worker picks up the new code. The file
# watcher runs only in the master (started after the initial fork).
# Enabling watch also turns on dev error reporting: fatal worker errors expose
# the exception class + stack trace to the client (HTTP 500 body / gRPC INTERNAL
# message). In production leave it off — the client sees a generic message and
# the full detail is logged server-side. (Surfaced to workers as FOLK_DEV_MODE.)
[dev]
watch = false                          # Enable file watcher + hot reload (+ dev error detail)
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
shutdown_timeout = "30s"               # Grace period to drain in-flight connections on shutdown (distinct from [server] shutdown_timeout)
max_request_size = "10mb"              # Max request body ("10mb", "512kb", or bytes). Not enforced when stream_request_body = true
stream_request_body = false            # Stream body to PHP (Folk::read; multipart via Folk::nextPart) instead of buffering. See streaming.md
stream_request_body_paths = []         # Restrict streaming to these paths ("*" suffix = prefix match). Empty = every path. Others stay buffered.
access_log = false                     # Log every request (method, URI, status, duration)
trusted_proxies = []                   # CIDR subnets for X-Forwarded-For extraction
h2c = false                            # HTTP/2 cleartext (without TLS)
# public_dir = "public"                # Serve matching files from this dir (nginx try_files) before dispatching to PHP; miss → PHP. .php and non-GET/HEAD always go to PHP. Relative to project root. Unset = disabled.

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
# required = false                     # true → a missing script or compile error ABORTS startup
                                       # (instead of skipping the hook with a WARN)

# =============================================================================
# Jobs Plugin
# =============================================================================
# Each backend ("driver") is declared as a connection that nests its own
# queues. The connection key IS the driver name, so there is ONE instance per
# driver type (two [jobs.connections.redis] sections = a TOML duplicate-key
# error). Different drivers run side by side.
#
# Drivers must be compiled into the build: default features are
# ["memory", "redis"]; every other driver is opt-in via folk.build.toml
# features — "embedded" (redb) and the managed brokers "rabbitmq", "sqs",
# "nats", "beanstalk", "kafka", "pubsub". A connection section whose driver is
# not compiled in makes the server fail fast at startup with an actionable
# message. Broker drivers hold each message in-flight until the job succeeds
# (ack) or terminally fails (nack → broker DLX/redelivery), so a crashed worker
# re-delivers (at-least-once); memory/redis/embedded remain at-most-once.

# In-memory driver — no persistence. The section only declares queues.
[jobs.connections.memory]
  [jobs.connections.memory.queues.default]
  concurrency = 4                      # Concurrent consumers for this queue
  max_retries = 3                      # Retries before DLQ or discard
  retry_delay = "1s"                   # Base delay between retries
  retry_backoff = "exponential"        # "exponential", "linear", "fixed"
  job_timeout = "60s"                  # Max job execution time ("0s" = unlimited)
  # dead_letter_queue = "failed"       # Queue name for failed jobs (omit to discard)
  priority = 10                        # Lower number = higher priority

# Redis driver.
# [jobs.connections.redis]
# host = "127.0.0.1"
# port = 6379
# username = ""                        # Redis ACL username (optional)
# password = ""
# db = 0
# tls = false                          # true → rediss:// scheme
# key_prefix = ""                      # Prefix applied to all queue keys
# pool_size = 8
# connect_timeout = "5s"
# command_timeout = "5s"
# url = ""                             # Full URL override; non-empty wins over the fields above
#   [jobs.connections.redis.queues.emails]
#   concurrency = 8
#   dead_letter_queue = "failed"

# Embedded driver (redb) — pure-Rust persistent ACID queue, single file.
# Requires the "embedded" Cargo feature in the build.
# [jobs.connections.embedded]
# path = "var/jobs.redb"
# durability = "eventual"              # "eventual" (fast) or "immediate" (fsync per commit)
#   [jobs.connections.embedded.queues.heavy]
#   concurrency = 2

# --- Managed-broker drivers (each requires its Cargo feature) ---

# RabbitMQ (AMQP 0-9-1). Feature: "rabbitmq". Manual ack/nack + prefetch.
# [jobs.connections.rabbitmq]
# host = "127.0.0.1"
# port = 5672
# vhost = "/"
# username = "guest"
# password = "guest"
# tls = false                          # true → amqps:// scheme
# prefetch = 10                        # Max unacked messages per consumer (QoS)
# dead_letter_exchange = ""            # x-dead-letter-exchange for nack'd jobs (empty = none)
# url = ""                             # Full amqp:// URL override; non-empty wins
#   [jobs.connections.rabbitmq.queues.emails]
#   concurrency = 8
# Note: delay is implemented via a deferred publish (no native delay); lost if
# the server crashes inside the delay window.

# AWS SQS. Feature: "sqs". Standard + FIFO (*.fifo) queues.
# [jobs.connections.sqs]
# region = "us-east-1"
# endpoint = ""                        # Override for LocalStack, e.g. http://localstack:4566
# access_key_id = ""                   # Empty → default AWS provider chain (env/profile/IAM)
# secret_access_key = ""
# visibility_timeout = 60              # Seconds a received message stays invisible
# wait_time_seconds = 20               # Long-poll wait (0–20)
#   [jobs.connections.sqs.queues.default]
#   concurrency = 4
# Note: native delay (DelaySeconds) is capped at 900s; terminal nack deletes the
# message — for native dead-lettering configure a redrive policy.

# NATS / JetStream. Feature: "nats". Durable pull consumers with ack.
# [jobs.connections.nats]
# url = "nats://127.0.0.1:4222"
# username = ""
# password = ""
# token = ""
# stream = "folk"                      # JetStream stream capturing "<stream>.<queue>" subjects
# max_deliver = 3                      # Attempts before the message is terminated
# ack_wait_secs = 60                   # Redelivery window if not acked
#   [jobs.connections.nats.queues.events]
#   concurrency = 4

# beanstalkd. Feature: "beanstalk". A queue name maps to a tube.
# [jobs.connections.beanstalk]
# host = "127.0.0.1"
# port = 11300
# ttr_secs = 60                        # time-to-run (visibility timeout)
# priority = 1024                      # Job priority (lower = higher)
#   [jobs.connections.beanstalk.queues.default]
#   concurrency = 4
# Note: native delay + priority; terminal failure buries the job for inspection.

# Apache Kafka. Feature: "kafka". A queue name maps to a topic; consumer group
# with manual offset commit (at-least-once).
# [jobs.connections.kafka]
# brokers = "localhost:9092"           # Comma-separated bootstrap servers
# group_id = "folk"
# security_protocol = ""               # e.g. "SASL_SSL" (empty = PLAINTEXT)
# sasl_mechanism = ""                  # e.g. "PLAIN"
# sasl_username = ""
# sasl_password = ""
#   [jobs.connections.kafka.queues.events]
#   concurrency = 4
# Note: depth is not reported (0); delay via deferred publish.

# Google Cloud Pub/Sub. Feature: "pubsub". A queue maps to a topic; the pull
# subscription is "<queue><subscription_suffix>".
# [jobs.connections.pubsub]
# project_id = "my-project"
# endpoint = ""                        # Emulator host, e.g. 127.0.0.1:8085 (empty = real Pub/Sub via ADC)
# credentials_file = ""                # Service-account JSON path (empty = ADC)
# subscription_suffix = "-folk"
# ack_deadline_secs = 60
#   [jobs.connections.pubsub.queues.events]
#   concurrency = 4
# Note: depth is not reported (0); delay via deferred publish.

# Addressing from PHP: a job's queue name may carry an optional connection
# prefix — "[<connection>.]<queue>" (e.g. ->onQueue("redis.emails")). A bare
# name (no prefix) resolves directly when it is unique across all connections;
# if the same bare name exists in more than one connection, it is ambiguous and
# jobs.push returns an error asking for a prefix. The prefix is split on the
# FIRST dot and is treated as a connection only when it BOTH matches a declared
# connection AND the remainder names an existing queue in it — so avoid naming a
# queue with a leading "<driver>." segment.

# =============================================================================
# gRPC Plugin
# =============================================================================
[grpc]
listen = "0.0.0.0:50051"              # Listening address. OPTIONAL: omit it for a client-only
                                       # deployment (only [grpc.clients.*], no server bound).
proto = []                             # Proto files/dirs for reflection + transcoding (empty = no reflection)
transcode = false                      # Decode proto↔native so PHP handlers use typed DTOs (default: raw passthrough)
# descriptor_set = "all.pb"            # Optional prebuilt FileDescriptorSet merged into the pool (for Any, etc.)
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

# --- gRPC clients: call upstream services from PHP ---------------------------
# Each [grpc.clients.<name>] is a named upstream. Its own `proto` builds its own
# descriptor pool (isolated per upstream). PHP calls it via a generated
# {Service}Client stub (folk:grpc:generate --client <name> / folk-grpc-gen
# --client <name>) and Folk::grpcClient(Stub::class). No protoc, no ext-grpc.
# A config with only [grpc.clients.*] and no [grpc] listen is a valid client-only
# deployment.
# STREAMING: a streaming RPC on the upstream reuses this SAME config — no extra
# keys. The stub extends GrpcStreamClient; drain a server-stream with foreach,
# feed a client-stream with an iterable, or do both for bidi. `deadline` bounds
# the whole stream (→ DEADLINE_EXCEEDED). See plugins/grpc.md#streaming.
# [grpc.clients.catalog]
# proto = ["proto/clients/catalog.proto"]   # This upstream's contract (files/dirs)
# address = "catalog.svc:50051"             # host:port; or a list ["a:1","b:2"] → round-robin LB
# deadline = "5s"                           # Default per-call deadline (a call may override it)
# descriptor_set = "catalog.pb"             # Optional prebuilt FDS merged into this client's pool
# [grpc.clients.catalog.retries]            # Retry ONLY transient (UNAVAILABLE), never a business status
# max_attempts = 3
# backoff = "100ms"                         # Exponential: backoff * 2^(n-1)
# [grpc.clients.catalog.tls]                # Upstream TLS/mTLS (distinct from the server listener)
# ca = "/path/to/ca.pem"                    # CA to verify the upstream (omit = system roots)
# cert = "/path/to/client.pem"              # Client cert for mTLS (with key)
# key = "/path/to/client-key.pem"
# domain = "catalog.internal"               # SNI / cert name override

# =============================================================================
# Metrics Plugin
# =============================================================================
[metrics]
listen = "0.0.0.0:9090"               # Listening address (scrape server runs in the master)
prefix = "folk"                        # Namespace prefix for custom metrics (folk_*)
metrics_path = "/metrics"              # Prometheus scrape endpoint
health_path = "/health"                # Liveness probe (always 200)
ready_path = "/ready"                  # Readiness probe (503 if not ready)
max_series = 10000                     # Cap on distinct series in the shared-memory segment;
                                       # new series past it increment folk_metrics_series_dropped

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
# restart = "always"                   # REQUIRED: "always", "on_failure", "never"
# max_restarts = 5                     # REQUIRED: max restart attempts
# restart_delay = "2s"                 # REQUIRED: delay between restarts
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
