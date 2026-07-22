# HTTP Plugin

Accepts HTTP connections and dispatches requests to PHP workers. Built on [hyper](https://hyper.rs/) and [axum](https://github.com/tokio-rs/axum).

## Features

- HTTP/1.1 server on hyper/axum
- Zero-copy dispatch to PHP workers via channels
- Request/response as structured `serde_json::Value`
- Graceful shutdown
- Configurable read/write timeouts (applied via tower-http)
- Configurable max request body size (human-readable: `"10mb"`, `"512kb"`)
- HTTP access logging (client IP, method, URI, status, duration, response bytes)
- Trusted proxies — correct X-Forwarded-For extraction behind load balancers
- TLS/SSL via rustls (optional feature `tls`, enabled by default)
- HTTP/2 cleartext (h2c) via hyper-util (optional feature `h2c`)
- Response compression: gzip, brotli, zstd, deflate (configurable algorithms and min size)
- Static file serving from a directory (`public_dir`, nginx `try_files` style)
- Active connections counter (via `http.connections` RPC)
- **Lua hook pipeline** — attach Lua scripts to request lifecycle events without writing Rust code

## Planned

- Per-route timeouts and body limits

## Configuration

```toml
[http]
listen = "0.0.0.0:8080"        # Listening address
read_timeout = "10s"            # Max time to read request body
write_timeout = "30s"           # Max time to write response (returns 504 on timeout)
max_request_size = "10mb"       # Max request body size ("10mb", "512kb", or integer bytes)
access_log = false              # Enable HTTP access logging
trusted_proxies = []            # Trusted CIDR subnets for X-Forwarded-For
h2c = false                     # Enable HTTP/2 cleartext (without TLS)
# public_dir = "public"         # Serve static files from this dir before dispatching to PHP (disabled by default)

# TLS — if set, the server listens on HTTPS (HTTP/2 via ALPN automatic)
# [http.tls]
# cert = "/path/to/cert.pem"
# key = "/path/to/key.pem"

# Response compression
# [http.compression]
# enabled = true
# algorithms = ["gzip", "br", "zstd"]   # in priority order
# min_size = 256                          # min response size to compress (bytes)
```

### Trusted Proxies

When running behind a load balancer or reverse proxy, configure `trusted_proxies` to correctly extract the real client IP from the `X-Forwarded-For` header:

```toml
[http]
trusted_proxies = ["10.0.0.0/8", "172.16.0.0/12", "192.168.0.0/16"]
```

Folk uses the **rightmost non-trusted** algorithm — it walks the X-Forwarded-For chain from right to left and returns the first IP that is not in a trusted subnet. This is the secure standard approach that prevents spoofing.

### Compression

Enable response compression to reduce bandwidth:

```toml
[http.compression]
enabled = true
algorithms = ["gzip", "br"]    # supported: gzip, br, zstd, deflate
min_size = "1kb"                # don't compress small responses
```

The server respects the client's `Accept-Encoding` header and selects the best matching algorithm from the configured list.

### Static files

Set `public_dir` to serve static assets straight from disk before a request reaches PHP — the same `try_files` behaviour you'd get from nginx or `php artisan serve`:

```toml
[http]
public_dir = "public"    # relative to the project root; unset (default) = disabled
```

- A request that maps to an existing file under `public_dir` is served from disk (with the correct `Content-Type`), so built assets (`/build/assets/app-*.css`, JS, images, fonts) never hit a PHP worker.
- A miss **falls through** to the PHP handler, so your framework routes keep working.
- `.php` files and non-`GET`/`HEAD` requests are **always** dispatched to PHP — the framework front controller (`public/index.php`) is never returned as source, and `/` is handled by the framework, not `public/index.html`.

Compression applies to static responses too. For heavy asset traffic you may still prefer a CDN or reverse proxy in front of Folk; `public_dir` removes the hard requirement for one.

## How It Works

1. HTTP request arrives at the Rust listener
2. Request is converted to a `serde_json::Value` (method, headers, body, URI)
3. Dispatched to an available PHP worker via channel
4. PHP handler processes the request and returns a response
5. Response is sent back to the client

The entire path is zero-copy between Rust and PHP — no JSON serialization on the hot path when using the dispatch loop.

## PHP Handler

In your worker script, register an HTTP handler:

```php
$loop = new \Folk\Sdk\Worker\WorkerLoop();

$loop->onHttp(function (array $request): array {
    return [
        'status' => 200,
        'headers' => ['Content-Type' => 'application/json'],
        'body' => json_encode(['hello' => 'world']),
    ];
});

$loop->run();
```

With Laravel, HTTP routing works automatically via the Folk service provider.

## Lua Hook Pipeline

Attach Lua scripts to points in the HTTP request lifecycle — before or after PHP, without writing or recompiling Rust code. Useful for rate limiting, auth checks, header manipulation, CORS, audit logging, and A/B routing.

### Hook events

| Event | When | Can short-circuit | Context available |
|-------|------|:-----------------:|-------------------|
| `request.before` | After HTTP parsing, before PHP dispatch | yes | method, path, query, headers, client_ip, request_id, extra |
| `request.error` | PHP returned error or exec_timeout fired | no | same + `error` string |
| `response.headers` | PHP response headers received, before body | yes | status, resp_headers |
| `response.after` | Full PHP response received, before sending | yes | status, resp_headers, body |

### Configuration

```toml
[[http.hooks]]
event = "request.before"
lua = "hooks/rate_limit.lua"
mode = "sync"           # "sync" (critical path) | "async" (fire-and-forget)
timeout_ms = 5          # sync only: abort hook after N ms; default 5
on_error = "fail_open"  # "fail_open" (skip + WARN) | "fail_closed" (→ 500)
```

Multiple `[[http.hooks]]` entries are allowed. On each event, all `sync` hooks run first (in declaration order), then all `async` hooks fire-and-forget. A short-circuiting sync hook stops subsequent sync hooks and PHP dispatch; async hooks still run with `ctx.short_circuited = true`.

### Writing Lua scripts

The script receives a `ctx` global table. Return `nil` (or nothing) to continue; return a table to short-circuit:

```lua
-- hooks/rate_limit.lua
if ctx.headers["x-api-key"] ~= "secret" then
    return { status = 429, body = "Too Many Requests" }
end
```

```lua
-- hooks/cors.lua  (response.headers event)
ctx.resp_headers["access-control-allow-origin"] = "*"
ctx.resp_headers["access-control-allow-methods"] = "GET, POST, OPTIONS"
```

```lua
-- hooks/inject.lua  (request.before — mutate before PHP sees it)
ctx.headers["x-forwarded-host"] = ctx.headers["host"]
ctx.extra["trace_start"] = "1"
```

**Context fields:**

| Field | Events | Mutable | Notes |
|-------|--------|:-------:|-------|
| `ctx.method` | request.* | no | `"GET"`, `"POST"`, … |
| `ctx.path` | request.* | no | URI path, e.g. `"/api/users"` |
| `ctx.query` | request.* | no | Raw query string without `?` |
| `ctx.client_ip` | request.* | no | Resolved client IP |
| `ctx.request_id` | request.* | no | UUID v7; `""` outside request |
| `ctx.headers` | request.* | yes (sync) | Mutations reach PHP |
| `ctx.extra` | request.* | yes (sync) | Arbitrary bag; mutations reach PHP |
| `ctx.error` | request.error | no | Error message or `"timeout"` |
| `ctx.status` | response.* | no | HTTP status code |
| `ctx.resp_headers` | response.* | yes (sync) | Mutations reach the client |
| `ctx.body` | response.after | yes (sync) | Full response body as string |
| `ctx.short_circuited` | all (async) | no | `true` if a prior sync hook short-circuited |

### Error handling

- `on_error = "fail_open"` (default) — if the Lua script errors, log WARN and continue pipeline.
- `on_error = "fail_closed"` — if the Lua script errors, return HTTP 500. Use for security-critical hooks (auth, API-key check) where a failing script must not silently pass requests through.

If a script file is missing or has a syntax error at startup, the hook is skipped with WARN and the server starts normally.

## Request ID

Every request is assigned a globally-unique id — a **UUID v7** (time-ordered and
unique across instances and restarts). When `access_log = true`, it is included
as the `request_id` field of each access-log line, so the Rust-side line and your
PHP application log of the same request share one id. Read it from PHP with the
`\Folk\Sdk\Folk::requestId()` facade (or the native `folk_request_id()` function):

```php
use Folk\Sdk\Folk;

Log::withContext(['request_id' => Folk::requestId()]);
```

`requestId()` returns `""` outside of a request, or when the Folk extension is not
loaded (e.g. in unit tests), so it is always safe to call.

> Concurrency model: each worker process handles **one request at a time**.
> Scale HTTP throughput with more worker processes — either the shared
> `[workers] count` pool, or a dedicated HTTP pool via `[http] workers = N`
> (see [Scaling](#scaling)).

## Scaling

By default the HTTP plugin runs in the shared worker pool alongside the other
plugins. To give HTTP its own process pool — sized independently of jobs or gRPC
concurrency — set `workers` in the `[http]` section:

```toml
[workers]
count = 4        # shared pool

[http]
workers = 8      # 8 dedicated HTTP-only processes
```

Each worker binds the listener with `SO_REUSEPORT`, so the kernel load-balances
accepted connections across the pool with no user-space dispatcher. With no
per-plugin `workers` anywhere, HTTP shares the `[workers] count` pool — identical
to earlier releases.
