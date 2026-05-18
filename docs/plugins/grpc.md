# gRPC Plugin

Native gRPC server with reflection support. Built on [tonic](https://github.com/hyperium/tonic).

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
