# Streaming

Folk can stream both directions of an HTTP exchange without buffering whole
bodies in memory:

- **Request streaming** — receive large uploads (raw or `multipart/form-data`)
  chunk by chunk. Folk replaces nginx as the upload buffer.
- **Response streaming** — send a response incrementally (SSE, large exports,
  proxied streams) with true HTTP chunked transfer.

There are two ways to use it:

1. **Low-level** — call `Folk::read()` / `Folk::nextPart()` / `Folk::writeHead()`
   directly in a handler. Works in any setup, framework or not. See
   [PHP API → Request body streaming](php-api.md#request-body-streaming).
2. **Framework-native** — the adapter wires streaming into the normal request /
   response lifecycle, so you keep using `$request->file()`, validation and
   framework streamed-response objects. This page documents that path.

Framework wiring status:

| Framework | Request (uploads) | Response | Response streaming trigger |
|-----------|:-----------------:|:--------:|----------------------------|
| **Laravel** | ✅ | ✅ | `StreamedResponse` |
| **Symfony** | ✅ | ✅ | `StreamedResponse` |
| **Spiral** | ✅ | ✅ | `X-Folk-Stream: yes` / unknown body size |
| **Yii 3** | ✅ | ✅ | `X-Folk-Stream: yes` / unknown body size |

---

## Enabling streaming

Streaming is **opt-in** and **per-path**, configured in `folk.toml`. Without it,
every request is buffered exactly as before — turning it on only affects the
paths you list.

```toml
[http]
stream_request_body = true
# Only these paths stream; every other path stays buffered (and keeps
# max_request_size enforcement). A pattern ending in "*" is a prefix match;
# otherwise the path must match exactly.
stream_request_body_paths = ["/upload", "/api/files/*"]
```

- If `stream_request_body_paths` is **empty** while `stream_request_body = true`,
  **every** path streams (global mode).
- On streamed paths, `max_request_size` is **not** enforced — cap the size in
  your application instead (see [Limits](#limiting-upload-size)).

!!! warning "Streamed paths are adapter-managed"
    On a streamed path the Laravel adapter consumes the body for you to build the
    request. Use `$request->file()` / `$request->input()` / `$request->getContent()`
    in your controller — **not** raw `Folk::read()` / `Folk::nextPart()`, which
    would find the stream already drained. The two modes are mutually exclusive
    on the same path.

---

## Laravel

### File uploads

List your upload route in `stream_request_body_paths` and write the controller
exactly as you always would — `$request->file()`, validation, `->store()` all
work. Folk parses the multipart stream in Rust and the adapter spools each file
part to a temporary file on disk (chunked, so the worker never holds the whole
file in memory), then hands you a normal `UploadedFile`.

```toml
[http]
stream_request_body = true
stream_request_body_paths = ["/profile/avatar"]
```

```php
Route::post('/profile/avatar', function (Illuminate\Http\Request $request) {
    $request->validate([
        'name'   => 'required|string|max:255',
        'avatar' => 'required|image|max:10240', // KB
    ]);

    $path = $request->file('avatar')->store('avatars');

    return response()->json([
        'name' => $request->input('name'),   // text field from the same form
        'path' => $path,
    ]);
});
```

A mixed form (text fields **and** files) works transparently: text parts become
`$request->input(...)`, file parts become `$request->file(...)`.

!!! note "Temp files are cleaned up automatically"
    The spooled temp file is deleted after the response is sent (even if the
    controller throws). `->store()` / `->move()` behave normally.

!!! info "Disk, not zero-copy"
    The upload is streamed to a temp file on disk and then made available as an
    `UploadedFile`, so `$request->file('x')->store('s3')` reads the finished file
    from disk and copies it to S3 — the same two-step flow as classic PHP
    uploads. Memory stays flat; the file does land on local disk first. True
    pass-through streaming (client → S3, no disk) is a planned opt-in.

### Limiting upload size

Because `max_request_size` is disabled on streamed paths, set a PHP-side cap in
`config/folk.php`. Exceeding it returns **HTTP 413** and cleans up the partial
temp file.

```php
// config/folk.php
return [
    // ...
    'streaming' => [
        // Default cap for any streamed body. 0 = unlimited.
        'max_request_bytes' => 0,

        // Optional per-path caps, matched against the request path.
        // A pattern ending in "*" is a prefix match; otherwise exact.
        'limits' => [
            '/api/files/*' => 1073741824, // 1 GiB
        ],
    ],
];
```

### Raw / JSON bodies

A streamed non-multipart body (e.g. a large JSON payload) is read into the
request content, so `$request->getContent()`, `$request->json()` and
`$request->input()` work as usual on a streamed path:

```php
Route::post('/ingest', function (Illuminate\Http\Request $request) {
    return response()->json(['received' => strlen($request->getContent())]);
});
```

### Streamed responses

Return a Symfony/Laravel `StreamedResponse` (or `StreamedJsonResponse`) and the
adapter pipes its output through Folk's chunked-response primitives instead of
buffering it — the client receives `Transfer-Encoding: chunked` and each
`echo` flushes as it is produced. No `folk.toml` flag is needed for responses.

```php
use Symfony\Component\HttpFoundation\StreamedResponse;

Route::get('/export', function () {
    return new StreamedResponse(function () {
        $handle = fopen('php://output', 'w');
        foreach (App\Models\Order::cursor() as $order) {
            fputcsv($handle, $order->toArray());
            flush(); // push this row to the client now
        }
        fclose($handle);
    }, 200, [
        'Content-Type'        => 'text/csv',
        'Content-Disposition' => 'attachment; filename="orders.csv"',
    ]);
});
```

This is ideal for CSV/NDJSON exports, Server-Sent Events, and proxying long
upstream streams.

---

## Symfony

Identical to Laravel — Symfony uses the same HttpFoundation request/response.
List the route in `stream_request_body_paths`, then use `$request->files->get()`
and `$request->request->get()`; return a `StreamedResponse` for chunked output.

Per-path size limits come from container parameters (or the `FOLK_STREAM_MAX_BYTES`
env var):

```yaml
# config/services.yaml
parameters:
    folk.streaming.max_request_bytes: 0
    folk.streaming.limits:
        '/api/files/*': 1073741824
```

## Spiral & Yii 3 (PSR-7)

Uploads work through standard PSR-7: on a streamed path the adapter populates
`$request->getUploadedFiles()` and `$request->getParsedBody()` from the stream.

```php
// Spiral / Yii 3 controller
$file = $request->getUploadedFiles()['avatar'] ?? null;   // UploadedFileInterface
$name = ($request->getParsedBody() ?? [])['name'] ?? null;
```

PSR-7 has no `StreamedResponse` class, so a streamed response is signalled one of
two ways:

- the response body has an **unknown size** (`getBody()->getSize() === null`) — a
  genuinely lazy/generator-backed stream; or
- the response carries the header **`X-Folk-Stream: yes`** — an explicit opt-in,
  handy when the body technically has a size but you still want chunked output
  (SSE, long responses):

```php
return $response
    ->withHeader('Content-Type', 'text/event-stream')
    ->withHeader('X-Folk-Stream', 'yes');   // adapter pipes the body to Folk::write
```

Otherwise the response is buffered (which also keeps `response.after` Lua hooks
working). The per-path size limit comes from the `FOLK_STREAM_MAX_BYTES` env var.

## Low-level API

Every adapter still exposes the raw primitives (`Folk::read()`,
`Folk::nextPart()`, `Folk::writeHead()/write()/end()`) for handlers that bypass
the framework — see [PHP API](php-api.md#request-body-streaming). On a path the
adapter already manages, use the framework request/response instead; the two are
mutually exclusive on the same path.
