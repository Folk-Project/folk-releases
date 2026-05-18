# Folk

**High-performance PHP application server powered by Rust.**

Folk replaces nginx + php-fpm with a single binary that handles HTTP, gRPC, background jobs, metrics, and managed processes — all with zero-copy communication between Rust and PHP.

## Features

- **HTTP server** — Built on hyper/axum, dispatches requests to PHP workers
- **Background jobs** — In-memory or Redis-backed queues with retry policies
- **gRPC server** — Native gRPC with reflection support
- **Prometheus metrics** — `/metrics` and `/health` endpoints out of the box
- **Process manager** — Supervised background processes with restart policies
- **ZTS multi-worker** — Multiple PHP worker threads in a single process (no fork)
- **Plugin architecture** — Only include what you need

## Quick Start

### 1. Install the extension

```bash
composer require folk/sdk
```

The SDK includes a pre-built `folk.so` extension for your platform. See [Installation](installation.md) for manual setup.

### 2. Create `folk.toml`

```toml
[workers]
script = "vendor/bin/folk-worker"
count = 4

[http]
listen = "0.0.0.0:8080"
```

### 3. Run

```bash
php vendor/bin/folk-worker
```

Your application is now serving HTTP on port 8080 with 4 worker threads.

## Laravel Integration

```bash
composer require folk/laravel
```

Folk integrates with Laravel automatically via a service provider. HTTP routes, job dispatching, and gRPC handlers work out of the box.

## Performance

Benchmarks on Docker (2 CPU, 512MB, 4 workers):

| Server | Raw JSON (req/s) | Laravel /ping (req/s) |
|--------|------------------:|----------------------:|
| **Folk** | **53,000** | **3,700** |
| Swoole | 73,000 | — |
| RoadRunner | 17,000 | — |
| FrankenPHP | 3,900 | — |

See [Benchmarks](benchmarks.md) for methodology and full results.

## Architecture

```
┌─────────────────────────────────────┐
│          PHP Process (PID 1)        │
│                                     │
│  ┌─────────┐  ┌──────────────────┐  │
│  │ Worker 1 │  │  Rust Runtime    │  │
│  │ (main)   │  │                  │  │
│  ├─────────┤  │  ┌────────────┐  │  │
│  │ Worker 2 │◄─┤  │ HTTP Plugin│  │  │
│  │ (ZTS)    │  │  ├────────────┤  │  │
│  ├─────────┤  │  │ Jobs Plugin│  │  │
│  │ Worker 3 │  │  ├────────────┤  │  │
│  │ (ZTS)    │  │  │ gRPC Plugin│  │  │
│  ├─────────┤  │  ├────────────┤  │  │
│  │ Worker 4 │  │  │ Metrics    │  │  │
│  │ (ZTS)    │  │  └────────────┘  │  │
│  └─────────┘  └──────────────────┘  │
└─────────────────────────────────────┘
     channels          tokio async
  (std::sync::mpsc)     runtime
```

Workers communicate with the Rust runtime via lock-free channels — no sockets, no serialization overhead.
