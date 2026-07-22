# Folk

**High-performance PHP application server powered by Rust.**

Folk replaces nginx + php-fpm with a single binary that handles HTTP, gRPC, background jobs, metrics, and managed processes — all with zero-copy communication between Rust and PHP.

💬 News and discussion: [Folk on Telegram](https://t.me/folk_poject)

## Features

- **HTTP server** — Built on hyper/axum, dispatches requests to PHP workers
- **Background jobs** — Pluggable queue backends (memory, Redis, embedded redb, and the RabbitMQ/SQS/NATS/Kafka/beanstalkd/Pub/Sub brokers) with retries, delays and priorities
- **gRPC server** — Native gRPC with reflection support
- **Prometheus metrics** — `/metrics` and `/health` endpoints out of the box
- **Process manager** — Supervised background processes with restart policies
- **Fork-after-warm workers** — a master warms PHP once, then forks N worker processes (crash isolation, force-kill, per-worker memory recycling)
- **Per-plugin worker pools** — scale HTTP, jobs and gRPC independently with a dedicated process pool each (`[http] workers = N`)
- **Streaming responses** — True chunked HTTP via `Folk::writeHead/write/end`, SSE support
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

See [Configuration](configuration.md) for all available options and [PHP API](php-api.md) for available native functions.

### 4. Run

```bash
php vendor/bin/folk-worker
```

Your application is now serving HTTP on port 8080 with 4 worker processes.

## Laravel

### 1. Install

```bash
pie install folk-project/ext-folk
composer require folk/laravel
```

Folk integrates with Laravel automatically via a service provider. HTTP routes, job dispatching, and gRPC handlers work out of the box. Between requests the adapter resets auth, database transactions, events, queue, temp uploads, the container's `scoped` instances, and Inertia's shared props (Octane-parity), so persistent workers don't leak state across requests.

### 2. Create `folk.toml`

```toml
[workers]
script = "vendor/bin/folk-server"
count = 4

[http]
listen = "0.0.0.0:8080"
public_dir = "public"   # serve built assets (Vite/Inertia) from disk; miss → Laravel
```

### 3. Run

```bash
php vendor/bin/folk-server
```

### Per-request state reset

Folk keeps the app booted across requests, so request-scoped state you mutate must be reset before the next request. The built-in resetters cover the framework; register your own for package/app state via `config/folk.php`:

```php
// config/folk.php
'resetters' => [
    App\Folk\MyStateResetter::class, // implements Folk\Sdk\Reset\ResettableInterface
],
```

## Spiral

### 1. Install

```bash
pie install folk-project/ext-folk
composer require folk/spiral
```

Folk integrates with Spiral Framework 3.x. HTTP pipeline, job processing, gRPC, and Cycle ORM cleanup work out of the box.

### 2. Create `folk.toml`

```toml
[workers]
script = "vendor/bin/folk-server"
count = 4

[http]
listen = "0.0.0.0:8080"
```

### 3. Run

```bash
php vendor/bin/folk-server
```

## Symfony

### 1. Install

```bash
pie install folk-project/ext-folk
composer require folk/symfony
```

Folk integrates with Symfony 6.4/7.x/8.x. HTTP kernel, services resetter, and Doctrine ORM cleanup work out of the box.

### 2. Create `folk.toml`

```toml
[workers]
script = "vendor/bin/folk-server"
count = 4

[http]
listen = "0.0.0.0:8080"
```

### 3. Run

```bash
php vendor/bin/folk-server
```

## Yii 3

### 1. Install

```bash
pie install folk-project/ext-folk
composer require folk/yii3
```

Folk integrates with Yii 3 via its native PSR-15 pipeline. HTTP handling, state resetter, and Cycle ORM cleanup work out of the box.

### 2. Create `folk.toml`

```toml
[workers]
script = "vendor/bin/folk-server"
count = 4

[http]
listen = "0.0.0.0:8080"
```

### 3. Run

```bash
php vendor/bin/folk-server
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

Folk uses a **fork-after-warm** model: a single-threaded **master** boots PHP +
your framework once, then forks N **worker processes** (the master only
supervises). The embedded Rust runtime lives in each worker:

- **Rust runtime** (tokio) runs in each worker, handling its I/O: HTTP, gRPC, job queues
- **PHP worker** (one per process) handles business logic — your Laravel/Symfony/plain PHP code; the in-process Rust↔PHP bridge is zero-copy (no sockets, no serialization)
- **Master** supervises: respawn on crash, force-kill on `exec_timeout`, RSS recycle; the metrics scrape server runs here over a shared-memory segment

| Component | Role |
|-----------|------|
| Master (PID 1) | Warms PHP, forks + supervises workers; runs master-only plugins (metrics scrape) |
| Workers 1–N (processes) | Each: own tokio + PHP, serve requests on `SO_REUSEPORT` |
| HTTP Plugin | Accepts HTTP requests, dispatches to the worker's PHP |
| Jobs Plugin | Queue backends: memory, Redis, embedded redb, and managed brokers (RabbitMQ/SQS/NATS/Kafka/beanstalkd/Pub/Sub) |
| gRPC Plugin | Native gRPC server with reflection |
| Metrics Plugin | Prometheus `/metrics` + `/health` |
| Process Plugin | Supervised background processes |

## Contributing

Found a bug or have an idea? See [Contributing](contributing.md) for how to report issues and propose features.
