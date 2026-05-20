# gRPC Plugin

Native gRPC server with reflection support. Built on [tonic](https://github.com/hyperium/tonic).

## Features

- Unary gRPC calls with dispatch to PHP workers
- Server reflection via proto files (grpcurl, Postman auto-discovery)
- Automatic proto import resolution
- gRPC framing and trailers
- Metadata passthrough to PHP

## Planned

- TLS/SSL (rustls)
- Max message size limits
- Server-wide and per-RPC timeouts
- HTTP/2 keepalive
- Max concurrent streams
- gRPC compression (gzip)
- gRPC Health Checking Protocol (grpc.health.v1)
- Server-side streaming

## Configuration

```toml
[grpc]
listen = "0.0.0.0:50051"                  # Listening address
proto = ["proto/greeter.proto"]            # Proto files for reflection
```

When `proto` is non-empty, gRPC server reflection is enabled — tools like `grpcurl` and Postman can discover services automatically.

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
grpcurl -plaintext -d '{"name": "World"}' \
    localhost:50051 greeter.Greeter/SayHello
```
