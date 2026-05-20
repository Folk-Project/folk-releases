# Metrics Plugin

Exposes Prometheus metrics and a health check endpoint.

## Features

- Prometheus `/metrics` endpoint (text exposition format)
- `/health` endpoint with per-plugin health checks (JSON)
- Plugins register their own metrics via `MetricsRegistry`
- Health checks aggregated from all plugins

## Planned

- `/ready` endpoint (Kubernetes readiness probe)
- Metric prefix (`folk_*`)
- Per-plugin metrics filtering (`[metrics.plugins]`)
- Custom metrics from config (counter, gauge, histogram)
- RPC for incrementing metrics from PHP (`metrics.increment`, `metrics.observe`)

## Configuration

```toml
[metrics]
listen = "0.0.0.0:9090"    # Listening address
```

## Endpoints

| Path | Description |
|------|-------------|
| `/metrics` | Prometheus metrics in text format |
| `/health` | Health check (returns `200 OK` or `503` with JSON status) |

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
| **core** | `folk_workers_total`, `folk_request_dispatch_total`, `folk_request_dispatch_duration_seconds`, `folk_worker_restarts_total`, `folk_request_queue_size` |
| **http** | `folk_http_requests_total`, `folk_http_request_duration_seconds`, `folk_http_active_connections`, `folk_http_errors_total` |
| **jobs** | `folk_jobs_pushed_total`, `folk_jobs_processed_total`, `folk_jobs_processing_duration_seconds`, `folk_jobs_queue_depth` |
| **grpc** | `folk_grpc_requests_total`, `folk_grpc_request_duration_seconds`, `folk_grpc_active_streams` |
| **process** | `folk_process_up`, `folk_process_restarts_total`, `folk_process_uptime_seconds` |
