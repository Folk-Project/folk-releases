# PHP API

Folk exposes native PHP functions via the extension and a `\Folk\Sdk\Folk` facade for the most common operations. All native functions are available globally once the extension is loaded.

---

## SDK Facade (`\Folk\Sdk\Folk`)

The facade wraps native functions and degrades gracefully (no-op) when the extension is not loaded — useful for running tests without the extension.

### `Folk::requestId(): string`

Returns the UUID v7 of the request currently being handled. Returns `""` outside of a request context or when the extension is not loaded.

```php
$id = \Folk\Sdk\Folk::requestId();
// → "019efdf3-503c-7bb3-a077-b501530fb2ad"
// → "" outside a request or without the extension
```

### `Folk::writeHead(int $status, array $headers = []): void`

Start a streaming response by sending the HTTP status code and headers. Must be called exactly once per request, before any `write()` call. After calling this, the return value of the PHP handler is ignored.

```php
\Folk\Sdk\Folk::writeHead(200, [
    'Content-Type' => 'text/event-stream',
    'Cache-Control' => 'no-cache',
]);
```

### `Folk::write(string $data): void`

Send a chunk of the response body. `writeHead()` must have been called first. May be called multiple times. Blocks until the chunk is accepted — provides backpressure for slow clients.

```php
\Folk\Sdk\Folk::write("data: hello\n\n");
```

### `Folk::end(): void`

Finish the streaming response. No more writes are possible after this. Return from the handler immediately after calling `end()`.

```php
\Folk\Sdk\Folk::end();
```

---

## Native functions

### `folk_version(): string`

Returns the version string of the loaded Folk extension.

```php
echo folk_version(); // "folk-ext 0.3.7"
```

### `folk_request_id(): string`

Returns the UUID v7 of the current request. Returns `""` when no request is in flight.

```php
$id = folk_request_id();
```

### `folk_is_worker_thread(): bool`

Returns `true` when called from a Folk ZTS worker thread. Returns `false` from the CLI or from code running outside the worker context (e.g., bootstrap, migrations).

```php
if (folk_is_worker_thread()) {
    // register request-scoped resources
}
```

### `folk_call(string $method, string $payload): string`

Call an RPC method registered by a plugin. `$payload` is a binary string (arbitrary bytes). Returns the response bytes as a binary string.

Typically used by adapters to talk to the Jobs plugin or custom RPC methods.

```php
$response = folk_call('jobs.push', $serialized_job);
```

### `folk_write_head(int $status, string $headers_json): void`

Low-level version of `Folk::writeHead()`. `$headers_json` is a JSON object `{"Header-Name": "value", ...}`.

```php
folk_write_head(200, json_encode(['Content-Type' => 'text/plain']));
```

### `folk_write(string $data): void`

Low-level version of `Folk::write()`. Send a body chunk.

```php
folk_write("chunk of data");
```

### `folk_write_end(): void`

Low-level version of `Folk::end()`. Finish the streaming response.

```php
folk_write_end();
```

### `folk_worker_run(string $dispatch_fn): void`

Run the zero-copy dispatch loop. Blocks until the server shuts down. `$dispatch_fn` must be a callable name with signature:

```php
function(string $method, array $params): array
```

Used internally by adapters (`bin/folk-server`). Not intended for direct use in application code.

### `folk_worker_ready(): bool`

Signal to the runtime that this worker has booted and is ready to accept requests. Returns `true` on the first call, `false` on subsequent calls. Called automatically by adapters.

### `folk_worker_recv(): ?array`

Block until a request arrives. Returns `[string $method, string $payload]` or `null` when the channel is closed (shutdown). Used by adapters that implement a manual dispatch loop.

### `folk_worker_send(string $result): void`

Send a serialized response back to the runtime. `$result` is raw bytes (typically JSON). Used by adapters that implement a manual dispatch loop.

### `folk_worker_send_error(string $message): void`

Signal an application-level error for the current request. The runtime will return a 502 to the client and log the message. Used by adapters that implement a manual dispatch loop.

---

## Streaming responses

By default, a PHP handler returns a complete response. Folk also supports true chunked streaming — useful for SSE, large downloads, or any response where you want to start sending before PHP finishes.

### Basic usage

```php
// Laravel route
Route::get('/stream', function () {
    \Folk\Sdk\Folk::writeHead(200, [
        'Content-Type' => 'text/plain',
        'X-My-Header'  => 'value',
    ]);

    \Folk\Sdk\Folk::write("first chunk\n");
    \Folk\Sdk\Folk::write("second chunk\n");
    \Folk\Sdk\Folk::end();
    // return value is ignored after writeHead()
});
```

Folk pipes the chunks directly to the client with `transfer-encoding: chunked`. No buffering on the Rust side.

### SSE (Server-Sent Events)

```php
Route::get('/events', function () {
    \Folk\Sdk\Folk::writeHead(200, [
        'Content-Type'      => 'text/event-stream',
        'Cache-Control'     => 'no-cache',
        'X-Accel-Buffering' => 'no',
    ]);

    foreach (generateEvents() as $event) {
        \Folk\Sdk\Folk::write("data: {$event}\n\n");
    }

    \Folk\Sdk\Folk::end();
});
```

### Request ID in streaming responses

`Folk::requestId()` returns the same UUID for the entire duration of the handler — including while calling `writeHead/write/end`. Use it to correlate logs or expose it to the client:

```php
Route::get('/events', function () {
    \Folk\Sdk\Folk::writeHead(200, [
        'Content-Type' => 'text/event-stream',
        'Cache-Control' => 'no-cache',
        'X-Request-ID' => \Folk\Sdk\Folk::requestId(), // expose to client
    ]);

    try {
        foreach (generateEvents() as $event) {
            \Folk\Sdk\Folk::write("data: {$event}\n\n");
        }
    } catch (\Throwable $e) {
        // requestId() is still valid here — same UUID
        Log::error('stream failed', [
            'request_id' => \Folk\Sdk\Folk::requestId(),
            'error'      => $e->getMessage(),
        ]);
    }

    \Folk\Sdk\Folk::end();
});
```

The same UUID also appears in Folk's Rust-side HTTP access log, so you can correlate server logs with client-visible `X-Request-ID` header.

### Note on hooks

If `response.after` hooks are configured in `folk.toml`, Folk buffers the full response body before running them. True streaming is only active when no `response.after` hooks are registered.

### Backward compatibility

Handlers that return a value without calling `writeHead()` continue to work exactly as before. Folk converts the return value to the streaming protocol internally.

---

## Request IDs

Every request gets a UUID v7 that is:
- **Globally unique** — across all workers, processes, and restarts
- **Time-ordered** — lexicographically sortable by creation time
- **Stable** — the same ID is available in both the PHP handler and the Rust HTTP access log

```php
// In your PHP handler
$id = \Folk\Sdk\Folk::requestId();
Log::info('handling request', ['request_id' => $id]);

// The same UUID appears in Folk's access log:
// time=2026-06-25T... method=GET path=/api/users status=200 request_id=019efdf3-...
```

Adapters (Laravel, Symfony, Spiral, Yii3) automatically inject the request ID into application logs via a Monolog processor — no manual wiring needed.

---

## Error handling

Folk distinguishes a **business outcome** (an expected result the application
decided on) from a **fatal error** (something broke). There are no
Folk-specific exception classes — you use native PHP exceptions and your
framework's error handling as-is.

### Business outcomes

- **HTTP** — your framework already renders them. `abort(404)`,
  `NotFoundHttpException`, `ValidationException`, etc. produce a normal response
  with the right status code; Folk passes it through unchanged. No Folk code
  required.
- **gRPC** — report the outcome via the context and return `null`:

  ```php
  use Folk\Sdk\Grpc\Context;

  public function GetUser(Context $context, GetUserRequest $req): ?UserReply
  {
      if (!$found) {
          $context->setStatus(5, 'user not found');   // canonical gRPC NOT_FOUND
          return null;
      }
      return $reply;
  }
  ```

  See the [gRPC plugin docs](plugins/grpc.md#status-codes) for the full code table.

### Fatal errors

An uncaught exception, a dispatch failure, or a malformed response is a fatal:

- **HTTP** → `500` (a worker that produces no response at all → `502`).
- **gRPC** → `INTERNAL (13)`.

The exception class and stack trace are exposed to the client **only in dev
mode** — enable `[dev] watch` in `folk.toml`, which Folk surfaces to workers via
the `FOLK_DEV_MODE` environment variable. In production the client receives a
generic message and the full detail is written to the server log only.
