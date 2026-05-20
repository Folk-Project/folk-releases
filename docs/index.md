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
pie install folk-project/ext-folk
```

See [Installation](installation.md) for Docker setup and building from source.

### 2. Install the SDK

```bash
composer require folk/sdk
```

### 3. Create `folk.toml`

```toml
[workers]
script = "vendor/bin/folk-worker"
count = 4

[http]
listen = "0.0.0.0:8080"
```

See [Configuration](configuration.md) for all available options.

### 4. Run

```bash
php vendor/bin/folk-worker
```

Your application is now serving HTTP on port 8080 with 4 worker threads.

## Laravel

### 1. Install the extension

```bash
pie install folk-project/ext-folk
```

### 2. Install the package

```bash
composer require folk/laravel
```

Folk integrates with Laravel automatically via a service provider. HTTP routes, job dispatching, and gRPC handlers work out of the box.

### 3. Create `folk.toml`

```toml
[workers]
script = "vendor/bin/folk-worker"
count = 4

[http]
listen = "0.0.0.0:8080"
```

See [Configuration](configuration.md) for all available options.

### 4. Run

```bash
php vendor/bin/folk-worker
```

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

Folk runs as a **single PHP process** with an embedded Rust runtime:

- **Rust runtime** (tokio) runs in a background thread, handling all I/O: HTTP, gRPC, job queues, metrics
- **PHP workers** (ZTS threads) handle business logic — your Laravel/Symfony/plain PHP code
- **Communication** via `std::sync::mpsc` channels — zero-copy, no sockets, no serialization overhead

| Component | Role |
|-----------|------|
| Worker 1 (main thread) | PHP worker + process entry point |
| Workers 2–N (ZTS threads) | Additional PHP workers |
| HTTP Plugin | Accepts HTTP requests, dispatches to workers |
| Jobs Plugin | In-memory or Redis job queues |
| gRPC Plugin | Native gRPC server with reflection |
| Metrics Plugin | Prometheus `/metrics` + `/health` |
| Process Plugin | Supervised background processes |
