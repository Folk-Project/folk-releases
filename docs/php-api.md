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

### `Folk::read(int $length = 8192): string`

Read up to `$length` bytes of the **request** body, blocking until data is available. Returns `""` at end-of-body. Only yields data when the HTTP plugin runs with `stream_request_body = true`; otherwise the body is in `$payload['body']` and this returns `""`. A single call may return fewer than `$length` bytes — loop until `""`.

```php
$handle = fopen('/var/uploads/file.bin', 'wb');
while (($chunk = \Folk\Sdk\Folk::read(65536)) !== '') {
    fwrite($handle, $chunk);
}
fclose($handle);
```

### `Folk::readAll(): string`

Read the entire request body, blocking until end-of-body. Returns `""` when there is no streaming body. Convenient when you don't need incremental processing.

```php
$body = \Folk\Sdk\Folk::readAll();
```

### `Folk::nextPart(): ?\Folk\Sdk\Http\Part`

Advance to the next part of a streamed `multipart/form-data` request. Returns the next [`Part`](#multipart-part), or `null` when there are no more parts (or the request is not a multipart streaming request, or the extension is not loaded). Any unread data of the current part is drained first, so it is safe to skip parts.

Yields parts only when the HTTP plugin runs with `stream_request_body = true` and the request `Content-Type` is `multipart/form-data` — Folk parses the boundary in Rust, so PHP never sees boundary bytes. See [Multipart streaming](#multipart-streaming).

```php
while ($part = \Folk\Sdk\Folk::nextPart()) {
    if ($part->isFile()) {
        $h = fopen('/var/uploads/' . basename($part->filename), 'wb');
        while (($chunk = $part->read(65536)) !== '') {
            fwrite($h, $chunk);
        }
        fclose($h);
    } else {
        $fields[$part->name] = $part->readAll();
    }
}
```

<a id="multipart-part"></a>
### `\Folk\Sdk\Http\Part`

One part of a streamed multipart request, returned by `Folk::nextPart()`.

- `string|null $name` — form field name.
- `string|null $filename` — file name (set for file parts, `null` for plain fields).
- `string|null $contentType` — the part's declared `Content-Type`.
- `isFile(): bool` — true when the part has a filename.
- `read(int $length = 8192): string` — next chunk of this part's body; `""` at the part's end.
- `readAll(): string` — the whole part body.

Parts are processed **sequentially**: read (or skip) the current part before calling `nextPart()` again. The read methods pull from Folk's current-part cursor — do not hold or read a `Part` after advancing.

---

## UUID helper (`\Folk\Sdk\Uuid`)

Dependency-free UUID generation, used by the framework queue adapters to stamp
job/message ids consistently with Folk's `request_id`.

### `Uuid::v7(): string`

A time-ordered RFC 9562 version 7 UUID (preferred — sorts by creation time).

### `Uuid::v4(): string`

A random RFC 4122 version 4 UUID.

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

### `folk_read(int $length = 8192): string`

Low-level version of `Folk::read()`. Read up to `$length` bytes of the request body, blocking until data is available; `""` at end-of-body.

```php
$chunk = folk_read(65536);
```

### `folk_read_all(): string`

Low-level version of `Folk::readAll()`. Read the entire request body.

```php
$body = folk_read_all();
```

### `folk_next_part(): ?string`

Low-level version of `Folk::nextPart()`. Returns a JSON object `{"name":...,"filename":...,"content_type":...}` for the next multipart part, or `null` when there are no more parts. Read the part body with `folk_part_read` / `folk_part_read_all`.

```php
$meta = folk_next_part(); // '{"name":"avatar","filename":"a.png","content_type":"image/png"}' | null
```

### `folk_part_read(int $length = 8192): string`

Read up to `$length` bytes of the current multipart part; `""` at the part's end.

### `folk_part_read_all(): string`

Read the entire current multipart part.

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

## Request body streaming

By default Folk reads the entire request body into memory before dispatching to PHP, and delivers it in `$payload['body']`. For large uploads this holds the whole body in RAM and is capped by `max_request_size` (413 otherwise).

With `stream_request_body = true` in `[http]`, Folk dispatches the request **before** the body is read. The body is delivered to PHP as a chunk stream, pulled via `Folk::read()` / `Folk::readAll()`. This lets you stream uploads straight to disk without buffering, and replaces nginx as an upload buffer.

```toml
[http]
stream_request_body = true
```

```php
// stream a large upload to disk, chunk by chunk
Route::post('/upload', function () {
    $handle = fopen(storage_path('app/upload.bin'), 'wb');
    $size = 0;
    while (($chunk = \Folk\Sdk\Folk::read(65536)) !== '') {
        $size += fwrite($handle, $chunk);
    }
    fclose($handle);
    return response()->json(['received' => $size]);
});
```

**Backpressure** is automatic: when PHP reads slowly, Folk stops reading the socket — the upload is paced by how fast PHP consumes it.

**Notes & limits:**

- Opt-in and **per-path** via `stream_request_body_paths` (empty = every path). When a path streams, `$payload['body']` is absent (`body_stream: true` is set instead) and PHP must use `Folk::read()`.
- `max_request_size` is **not** enforced in this mode — the application controls how much it reads.
- `read_timeout` (buffered-read timeout) does not apply; the overall ceiling is `[workers] exec_timeout`.
- If the client disconnects mid-upload, `Folk::read()` returns `""` (EOF) — validate `Content-Length` yourself if integrity matters.
- The code above is the **low-level** path. On **Laravel** you can instead let the adapter wire streaming into `$request->file()` / streamed responses — see [Streaming → Laravel](streaming.md#laravel). On a streamed path the adapter consumes the body, so use `$request`, not raw `Folk::read()`. Symfony/Spiral/Yii 3 still use the low-level API.

<a id="multipart-streaming"></a>
### Multipart form data

When `stream_request_body = true` and the request is `multipart/form-data`, Folk parses the boundary in Rust (no boundary bytes reach PHP) and hands you parts one at a time via `Folk::nextPart()`. File parts can be streamed to disk without buffering, exactly like raw bodies:

```php
Route::post('/upload', function () {
    $fields = [];
    while ($part = \Folk\Sdk\Folk::nextPart()) {
        if ($part->isFile()) {
            $h = fopen(storage_path('app/' . basename($part->filename)), 'wb');
            while (($chunk = $part->read(65536)) !== '') {
                fwrite($h, $chunk);
            }
            fclose($h);
        } else {
            $fields[$part->name] = $part->readAll();
        }
    }
    return response()->json(['fields' => $fields]);
});
```

Parts arrive in transmission order (fields and files interleaved as the client sent them). Process each part before calling `nextPart()` again — see [`Part`](#multipart-part). Backpressure and per-part streaming work the same as raw bodies. This is the low-level path; on **Laravel** the adapter turns streamed multipart parts into normal `$request->file()` uploads — see [Streaming → Laravel file uploads](streaming.md#file-uploads).

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
