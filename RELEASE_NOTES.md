### What's new

Streaming wired into the Laravel request/response lifecycle ([#73](https://github.com/Folk-Project/folk-releases/issues/73)).

Previous releases added the streaming primitives — raw body (`Folk::read`),
multipart parsing (`Folk::nextPart`), and chunked responses (`Folk::write`) —
but using them meant writing low-level handlers against `Folk::*`. The
framework still saw an empty-body request and buffered every streamed response.
This release connects streaming to the **normal Laravel lifecycle**: keep using
`$request->file()`, validation and `StreamedResponse`.

**Per-path activation.** Streaming is now opt-in per path, so enabling it for an
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

This is **Laravel-first**. Symfony, Spiral and Yii 3 keep using the low-level
`Folk::*` API until their adapters are wired in a later release. See the new
[Streaming guide](https://folk-project.github.io/folk-releases/streaming/).

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
| folk-sdk | 0.3.5 | packagist |
| folk/laravel | 0.3.6 | packagist |
| folk/spiral | 0.1.2 | packagist |
| folk/symfony | 0.1.2 | packagist |
| folk/yii3 | 0.1.1 | packagist |
