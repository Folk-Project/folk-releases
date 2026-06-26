### What's new

Streaming wired into the Symfony, Spiral and Yii 3 adapters ([#73](https://github.com/Folk-Project/folk-releases/issues/73)).

The previous release brought framework-native streaming to Laravel. This one
finishes the job for the remaining adapters — uploads and streamed responses now
flow through the normal request/response lifecycle everywhere. No extension
rebuild is needed: this is a pure-PHP release (the streaming primitives already
ship in the extension).

**Symfony** behaves exactly like Laravel (same HttpFoundation): list the route in
`stream_request_body_paths`, then use `$request->files->get()` /
`$request->request->get()` and return a `StreamedResponse` for chunked output.
Per-path size limits come from container parameters or `FOLK_STREAM_MAX_BYTES`.

**Spiral & Yii 3** (PSR-7): on a streamed path the adapter populates
`$request->getUploadedFiles()` and `$request->getParsedBody()` from the stream,
so controllers use standard PSR-7:

```php
$file = $request->getUploadedFiles()['avatar'] ?? null;   // UploadedFileInterface
$name = ($request->getParsedBody() ?? [])['name'] ?? null;
```

PSR-7 has no `StreamedResponse` class, so a response is streamed when its body
has an unknown size **or** it carries an explicit `X-Folk-Stream: yes` header —
handy for SSE and long responses whose body size is otherwise known:

```php
return $response
    ->withHeader('Content-Type', 'text/event-stream')
    ->withHeader('X-Folk-Stream', 'yes');   // adapter pipes the body to Folk::write
```

All the Laravel behaviour carries over: multipart parts spool to temp files
chunk by chunk and are cleaned up after the response, a per-path limit returns
HTTP 413, and buffered/known-size responses keep working (including
`response.after` Lua hooks).

This closes #73 — streaming is now wired into every Folk framework adapter. See
the [Streaming guide](https://folk-project.github.io/folk-releases/streaming/).

---

### Streaming wired into the Laravel lifecycle ([#73](https://github.com/Folk-Project/folk-releases/issues/73))

Earlier releases added the streaming primitives — raw body (`Folk::read`),
multipart parsing (`Folk::nextPart`), and chunked responses (`Folk::write`) —
but using them meant writing low-level handlers against `Folk::*`. The framework
still saw an empty-body request and buffered every streamed response. This
connected streaming to the **normal Laravel lifecycle**: keep using
`$request->file()`, validation and `StreamedResponse`.

**Per-path activation.** Streaming is opt-in per path, so enabling it for an
upload route does not break normal form/JSON routes:

```toml
[http]
stream_request_body = true
stream_request_body_paths = ["/upload", "/api/files/*"]   # "*" suffix = prefix match
```

Paths not listed stay buffered (and keep `max_request_size` enforcement). An
empty list keeps the previous global behaviour.

**File uploads.** On a streamed path, multipart parts are spooled to temp files
chunk by chunk (the worker never holds the whole file in memory) and handed to
the controller as ordinary uploads — write your route exactly as before:

```php
Route::post('/profile/avatar', function (Illuminate\Http\Request $request) {
    $request->validate([
        'name'   => 'required|string',
        'avatar' => 'required|image|max:10240',
    ]);
    return response()->json([
        'name' => $request->input('name'),            // text field
        'path' => $request->file('avatar')->store(),  // file part
    ]);
});
```

Temp files are deleted after the response is sent (even on exceptions). Because
`max_request_size` is disabled on streamed paths, set a PHP-side cap in
`config/folk.php` — exceeding it returns HTTP 413:

```php
'streaming' => [
    'max_request_bytes' => 0,                       // 0 = unlimited
    'limits' => ['/api/files/*' => 1073741824],     // 1 GiB on these paths
],
```

**Streamed responses.** Return a `StreamedResponse` / `StreamedJsonResponse` and
the adapter pipes its output through Folk's chunked-response primitives instead
of buffering it — ideal for CSV/NDJSON exports, SSE, and proxied streams.

Also: urlencoded form bodies are now parsed into POST parameters on buffered
routes.

### Versions

| Package | Version | Type |
|---------|---------|------|
| folk-api | 0.2.11 | crates.io |
| folk-core | 0.3.10 | crates.io |
| folk-ext | 0.3.10 | crates.io |
| folk-plugin-http | 0.4.4 | crates.io |
| folk-plugin-grpc | 0.2.7 | crates.io |
| folk-plugin-jobs | 0.3.4 | crates.io |
| folk-plugin-metrics | 0.2.3 | crates.io |
| folk-plugin-process | 0.2.4 | crates.io |
| folk-builder | 0.2.7 | crates.io |
| folk-sdk | 0.3.6 | packagist |
| folk/laravel | 0.3.6 | packagist |
| folk/spiral | 0.1.3 | packagist |
| folk/symfony | 0.1.3 | packagist |
| folk/yii3 | 0.1.2 | packagist |
