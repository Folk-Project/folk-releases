# Metrics Plugin

Exposes Prometheus metrics and a health check endpoint.

## Configuration

```toml
[metrics]
listen = "0.0.0.0:9090"    # Listening address
```

## Endpoints

| Path | Description |
|------|-------------|
| `/metrics` | Prometheus metrics in text format |
| `/health` | Health check (returns `200 OK` with JSON status) |

## Scraping with Prometheus

```yaml
# prometheus.yml
scrape_configs:
  - job_name: folk
    static_configs:
      - targets: ['app:9090']
```
