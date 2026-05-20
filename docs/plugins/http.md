# HTTP Plugin

Accepts HTTP connections and dispatches requests to PHP workers. Built on [hyper](https://hyper.rs/) and [axum](https://github.com/tokio-rs/axum).

## Features

- HTTP/1.1 server on hyper/axum
- Zero-copy dispatch to PHP workers via channels
- Request/response as structured `serde_json::Value`
- Graceful shutdown
- Configurable read/write timeouts

## Planned

- TLS/SSL (HTTPS via rustls)
- HTTP/2 (h2c)
- Gzip/Brotli/Zstd compression
- HTTP access logging
- Configurable max request size
- Trusted proxies (X-Forwarded-For)
- Static file serving
- CORS

## Configuration

```toml
[http]
listen = "0.0.0.0:8080"    # Listening address
read_timeout = "10s"        # Max time to read request body
write_timeout = "30s"       # Max time to write response
```

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
