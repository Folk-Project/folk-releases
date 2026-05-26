# gRPC Plugin

Native gRPC server with reflection and health checking. Built on [tonic](https://github.com/hyperium/tonic).

## Features

- Unary gRPC calls with dispatch to PHP workers
- Server reflection via proto files (grpcurl, Postman auto-discovery)
- Automatic proto import resolution
- gRPC Health Checking Protocol (grpc.health.v1)
- TLS/SSL via rustls
- HTTP/2 keepalive
- Max message size limits (human-readable: `"4mb"`)
- Server-wide RPC timeout
- Max concurrent streams
- gRPC compression (gzip)
- Metadata passthrough to PHP
- 6 Prometheus metrics

## Planned

- Server-side streaming
- Per-RPC timeouts

## Configuration

```toml
[grpc]
listen = "0.0.0.0:50051"                  # Listening address
proto = ["proto/greeter.proto"]            # Proto files for reflection
max_recv_message_size = "4mb"              # Max incoming message size
max_send_message_size = "4mb"              # Max outgoing message size
timeout = "30s"                            # Server-wide RPC timeout
max_concurrent_streams = 200               # HTTP/2 concurrent streams limit
compression = false                        # Enable gzip compression

# HTTP/2 keepalive
# [grpc.keepalive]
# interval = "60s"                         # PING interval
# timeout = "20s"                          # PING timeout before disconnect

# TLS — if set, the server listens on secure gRPC
# [grpc.tls]
# cert = "/path/to/cert.pem"
# key = "/path/to/key.pem"
```

When `proto` is non-empty, gRPC server reflection is enabled — tools like `grpcurl` and Postman can discover services automatically. The gRPC Health Checking Protocol is always enabled.

### Configuration reference

| Option | Type | Default | Description |
|--------|------|---------|-------------|
| `listen` | string | `"0.0.0.0:50051"` | Listening address |
| `proto` | array | `[]` | Proto files for reflection |
| `max_recv_message_size` | size | `"4mb"` | Max incoming message |
| `max_send_message_size` | size | `"4mb"` | Max outgoing message |
| `timeout` | duration | — | Server-wide RPC timeout |
| `max_concurrent_streams` | int | — | HTTP/2 stream limit |
| `compression` | bool | `false` | Enable gzip compression |

### Keepalive

```toml
[grpc.keepalive]
interval = "60s"     # Send HTTP/2 PING frames at this interval
timeout = "20s"      # Close connection if PONG not received
```

### TLS

```toml
[grpc.tls]
cert = "/etc/ssl/certs/server.crt"
key = "/etc/ssl/private/server.key"
```

Uses rustls (no OpenSSL dependency). When TLS is configured, the server automatically supports HTTP/2 via ALPN.

### Compression

When `compression = true`, the server:

- Decompresses incoming messages with gzip (`grpc-encoding: gzip`)
- Compresses outgoing messages if client sends `grpc-accept-encoding: gzip`
- Sets `grpc-encoding: gzip` response header when compressing

## Health Checking

The gRPC Health Checking Protocol (`grpc.health.v1.Health`) is always enabled:

```bash
grpcurl -plaintext localhost:50051 grpc.health.v1.Health/Check
# {"status": "SERVING"}
```

## PHP Handler

Register a gRPC handler in your worker script:

```php
$loop = new \Folk\Sdk\Worker\WorkerLoop();

$loop->onGrpc('greeter.Greeter/SayHello', function (array $request): array {
    return ['message' => 'Hello, ' . $request['name'] . '!'];
});

$loop->run();
```

## Testing

```bash
# List services (reflection)
grpcurl -plaintext localhost:50051 list

# Call a method
grpcurl -plaintext -d '{"name": "World"}' \
    localhost:50051 greeter.Greeter/SayHello

# Health check
grpcurl -plaintext localhost:50051 grpc.health.v1.Health/Check
```

## Metrics

The gRPC plugin registers the following Prometheus metrics (via the metrics plugin):

| Metric | Type | Labels | Description |
|--------|------|--------|-------------|
| `folk_grpc_requests_total` | Counter | `service`, `method`, `grpc_status` | Total gRPC calls |
| `folk_grpc_request_duration_seconds` | Histogram | `service`, `method` | Processing time |
| `folk_grpc_message_sent_bytes` | Histogram | `service`, `method` | Outgoing message size |
| `folk_grpc_message_received_bytes` | Histogram | `service`, `method` | Incoming message size |
| `folk_grpc_active_streams` | Gauge | — | Active gRPC streams |
| `folk_grpc_errors_total` | Counter | `service`, `method`, `type` | Errors by type |
