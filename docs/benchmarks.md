# Benchmarks

All benchmarks run on Docker with **2 CPU, 512MB RAM, 4 workers**.

## Raw JSON (no framework)

Minimal PHP handler returning `{"status":"ok"}`.

**Tool:** `wrk -t4 -c100 -d30s`

| Server | Req/sec | Errors |
|--------|--------:|--------|
| Swoole | 73,385 | 6K socket errors |
| **Folk** | **53,220** | **0** |
| RoadRunner | 17,165 | 0 |
| FrankenPHP | 3,892 | 0 |

## Laravel /ping

Laravel application returning a JSON response from a route.

**Tool:** `wrk -t4 -c50 -d30s`

| Server | Req/sec |
|--------|--------:|
| **Folk** | **3,703** |
| Swoole | 221* |

\* Swoole benchmark image may be misconfigured.

## Methodology

- All tests use the same hardware and Docker resource limits
- `wrk` runs from the host machine against the Docker container
- Each test runs for 30 seconds after a warm-up period
- Folk uses 4 forked worker processes
- Swoole, RoadRunner, and FrankenPHP use their default recommended configurations with 4 workers
