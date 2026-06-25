### What's new

Request body streaming ([#70](https://github.com/Folk-Project/folk-releases/issues/70)).

Folk can now stream the **request** body to PHP chunk by chunk instead of
buffering the whole thing in memory — the request-side counterpart to response
streaming. This lets you accept large uploads (and stream them straight to disk)
without holding them in RAM or hitting `max_request_size`, so Folk can replace
nginx as an upload buffer.

- **Opt-in via `[http] stream_request_body = true`** (default `false`). When
  off, behaviour is unchanged — the body is buffered into `$payload['body']`.
- When on, Folk dispatches the request **before** reading the body and delivers
  it as a chunk stream. PHP pulls it with `Folk::read(int $length = 8192)` /
  `Folk::readAll()` (low-level: `folk_read` / `folk_read_all`).
- **Automatic backpressure**: a slow PHP reader paces the upload — Folk stops
  reading the socket until PHP consumes more.
- `max_request_size` is not enforced in streaming mode; the application controls
  how much it reads.

```php
Route::post('/upload', function () {
    $handle = fopen(storage_path('app/upload.bin'), 'wb');
    while (($chunk = \Folk\Sdk\Folk::read(65536)) !== '') {
        fwrite($handle, $chunk);
    }
    fclose($handle);
    return response()->json(['ok' => true]);
});
```

This is Level 1 (raw streaming). `multipart/form-data` parsing and framework
adapter integration (PSR-7 streamed body, uploaded files, per-route activation)
are tracked in [#73](https://github.com/Folk-Project/folk-releases/issues/73).

---

### Worker response contract + error model ([#71](https://github.com/Folk-Project/folk-releases/issues/71), [#72](https://github.com/Folk-Project/folk-releases/issues/72)).

A worker now finalizes a request in one of three explicit shapes — a **verbatim
return value**, a **stream**, or a **fatal error** — instead of forcing every
response through the HTTP `{status, headers, body}` shape. This fixes two
regressions and adds a clean error model.

- **gRPC and jobs return values are preserved verbatim.** Previously a gRPC
  response was coerced into the HTTP response shape and lost, so every call
  failed with `INTERNAL (13)`. gRPC works again, and so do jobs.

- **Worker errors return their real status — never a silent `200`.** A fatal
  (uncaught exception, dispatch failure, malformed response) now yields HTTP
  `500` / gRPC `INTERNAL (13)`; a worker that produces no response yields `502`.

- **gRPC business status codes.** A handler reports a business outcome with
  `$context->setStatus($code, $message)` (standard server-side gRPC idiom) and
  returns `null`. The code is a canonical `google.rpc.Code`:

  | meaning | gRPC | HTTP |
  |---|---|---|
  | invalid_argument | 3 | 400 |
  | unauthenticated | 16 | 401 |
  | permission_denied | 7 | 403 |
  | not_found | 5 | 404 |
  | already_exists | 6 | 409 |
  | resource_exhausted | 8 | 429 |
  | unimplemented | 12 | 501 |
  | unavailable | 14 | 503 |
  | internal | 13 | 500 |

  HTTP business errors need no Folk code — the framework already renders them
  (`abort(404)`, `ValidationException`, …) and they travel as a normal return.

- **Dev-gated error detail.** Fatal responses include the exception class and
  stack trace only when dev mode is on (`[dev] watch`, surfaced to workers via
  the `FOLK_DEV_MODE` env var). In production the client gets a generic message;
  the full detail is always logged server-side.

- **Response streaming** (shipping in this release): stream large or
  server-sent-event responses directly with `folk_write_head` / `folk_write` /
  `folk_write_end`, exposed in the SDK as `Folk::writeHead/write/end`.

There are no Folk-specific exception classes: PHP exceptions and framework error
handling are used as-is.

**Minor BC (SDK):** `GrpcModeHandler::call()` now returns `?string` (returns
`null` when a business status is set).

**PHP usage:**
```php
// gRPC business outcome
public function GetUser(Context $ctx, GetUserRequest $req): ?UserReply {
    if (!$found) { $ctx->setStatus(5, 'user not found'); return null; } // NOT_FOUND
    return $reply;
}

// Streaming response
\Folk\Sdk\Folk::writeHead(200, ['Content-Type' => 'text/event-stream']);
foreach ($events as $event) {
    \Folk\Sdk\Folk::write("data: {$event}\n\n");
}
\Folk\Sdk\Folk::end();
```

### Versions

| Package | Version | Type |
|---------|---------|------|
| folk-api | 0.2.10 | crates.io |
| folk-core | 0.3.9 | crates.io |
| folk-ext | 0.3.9 | crates.io |
| folk-plugin-http | 0.4.2 | crates.io |
| folk-plugin-grpc | 0.2.7 | crates.io |
| folk-plugin-jobs | 0.3.4 | crates.io |
| folk-plugin-metrics | 0.2.3 | crates.io |
| folk-plugin-process | 0.2.4 | crates.io |
| folk-builder | 0.2.6 | crates.io |
| folk-sdk | 0.3.3 | packagist |
| folk/laravel | 0.3.5 | packagist |
| folk/spiral | 0.1.2 | packagist |
| folk/symfony | 0.1.2 | packagist |
| folk/yii3 | 0.1.1 | packagist |
