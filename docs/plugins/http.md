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
- Active connections counter (via `http.connections` RPC)

## Planned

- Static file serving
- CORS
- Per-route timeouts and body limits
- Rate limiting

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

## Request ID

Every request is assigned a unique, monotonic id. Read it from PHP with the
`\Folk\Sdk\Folk::requestId()` facade (or the native `folk_request_id()` function)
to correlate your application logs with Folk's Rust-side access log:

```php
use Folk\Sdk\Folk;

Log::withContext(['request_id' => Folk::requestId()]);
```

`requestId()` returns `0` outside of a request, or when the Folk extension is not
loaded (e.g. in unit tests), so it is always safe to call.

> Concurrency note: the `[workers] max_concurrent_per_worker` setting currently
> supports only `1` (one request per worker at a time). Values `> 1` are reserved
> for a future async runtime and are clamped to `1` with a warning.
