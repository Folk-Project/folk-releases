# Metrics Plugin

Exposes Prometheus metrics, health checks, and readiness probes. Supports custom metrics from config and RPC.

## Features

- Prometheus `/metrics` endpoint (text exposition format)
- `/health` liveness probe (always 200 if process alive)
- `/ready` readiness probe (runs health checks, 503 if not ready)
- Configurable endpoint paths
- Metric prefix (`folk_*` by default)
- Per-plugin metric filtering via `[metrics.plugins]`
- Declarative custom metrics from config (counter, gauge, histogram)
- RPC for manipulating custom metrics from PHP (`metrics.increment`, `metrics.observe`, `metrics.set`)
- `metrics.render` RPC for admin access to Prometheus snapshot

## Configuration

```toml
[metrics]
listen = "0.0.0.0:9090"       # Listening address
prefix = "folk"                 # Namespace prefix for custom metrics (folk_*)
metrics_path = "/metrics"       # Prometheus scrape endpoint path
health_path = "/health"         # Liveness probe path
ready_path = "/ready"           # Readiness probe path

# Per-plugin metric filtering (omit section = all enabled)
# [metrics.plugins]
# core = true       # folk_workers_*, folk_request_*, folk_boot_*
# http = true        # folk_http_*
# jobs = true        # folk_jobs_*
# grpc = false       # folk_grpc_* — disabled
# process = true     # folk_process_*
```

### Custom Metrics

Declare metrics in config — PHP can manipulate them via RPC without writing Rust code:

```toml
[[metrics.collectors]]
name = "app_requests_total"
type = "counter"
help = "Total application requests"
labels = ["method", "endpoint"]

[[metrics.collectors]]
name = "request_duration_seconds"
type = "histogram"
help = "Request processing duration"
buckets = [0.01, 0.05, 0.1, 0.5, 1.0, 5.0]

[[metrics.collectors]]
name = "queue_depth"
type = "gauge"
help = "Current queue depth"
```

Supported types: `counter`, `gauge`, `histogram`.

The `prefix` is prepended to custom metric names: with `prefix = "folk"`, a collector named `app_requests_total` becomes `folk_app_requests_total` in Prometheus output.

## Endpoints

| Path | Description |
|------|-------------|
| `/metrics` | Prometheus metrics in text exposition format |
| `/health` | Liveness probe — always 200 if process is alive |
| `/ready` | Readiness probe — 200 if all health checks pass, 503 otherwise |

### Kubernetes probes

```yaml
livenessProbe:
  httpGet:
    path: /health
    port: 9090
readinessProbe:
  httpGet:
    path: /ready
    port: 9090
```

`/health` is a simple liveness check — returns 200 as long as the process is running.

`/ready` runs all registered health checks (from all plugins) and returns:

- `200 OK` with `{"status":"ready","checks":{...}}` when all pass
- `503 Service Unavailable` with `{"status":"not_ready","checks":{...}}` when any check fails

## Plugin Filtering

Disable specific plugin metrics without removing the plugin:

```toml
[metrics.plugins]
grpc = false    # hide folk_grpc_* from /metrics output
```

If the `[metrics.plugins]` section is omitted, all metrics are included. Only explicitly set `false` disables a plugin's metrics.

## RPC Methods

| Method | Payload | Description |
|--------|---------|-------------|
| `metrics.increment` | `{name, labels?, value?}` | Increment a custom counter (default value=1) |
| `metrics.observe` | `{name, labels?, value}` | Observe a histogram value |
| `metrics.set` | `{name, labels?, value}` | Set a gauge value |
| `metrics.render` | — | Return Prometheus text snapshot |

### PHP usage

```php
// Increment a counter
folk_call('metrics.increment', json_encode([
    'name' => 'app_requests_total',
    'labels' => ['GET', '/api/users'],
]));

// Observe histogram
folk_call('metrics.observe', json_encode([
    'name' => 'request_duration_seconds',
    'value' => 0.123,
]));

// Set gauge
folk_call('metrics.set', json_encode([
    'name' => 'queue_depth',
    'value' => 42,
]));
```

## Scraping with Prometheus

```yaml
# prometheus.yml
scrape_configs:
  - job_name: folk
    static_configs:
      - targets: ['app:9090']
```

## Metrics by Plugin

Each plugin registers its own metrics. Available metrics depend on which plugins are included in your build:

| Plugin | Metrics |
|--------|---------|
| **grpc** | `folk_grpc_requests_total`, `folk_grpc_request_duration_seconds`, `folk_grpc_message_sent_bytes`, `folk_grpc_message_received_bytes`, `folk_grpc_active_streams`, `folk_grpc_errors_total` |
| **jobs** | `folk_jobs_pushed_total`, `folk_jobs_processed_total`, `folk_jobs_processing_duration_seconds`, `folk_jobs_retries_total`, `folk_jobs_dead_letter_total`, `folk_jobs_active` |
| **core** | `folk_workers_total`, `folk_request_dispatch_total`, `folk_request_dispatch_duration_seconds`, `folk_worker_restarts_total`, `folk_request_queue_size` (planned) |
| **http** | `folk_http_requests_total`, `folk_http_request_duration_seconds`, `folk_http_active_connections`, `folk_http_errors_total` (planned) |
| **process** | `folk_process_up`, `folk_process_restarts_total`, `folk_process_uptime_seconds` (planned) |
