### What's new

- **folk-api v0.2.8** — `ResponseChunk` enum (Headers/Body/End), `Executor::execute_streamed()` default method, `value_to_chunks()` helper ([#35](https://github.com/Folk-Project/folk-releases/issues/35))

- **folk-core/folk-ext v0.3.7** — full streaming protocol replaces the single-shot oneshot reply ([#35](https://github.com/Folk-Project/folk-releases/issues/35))
  - `WorkerPool::dispatch_streamed()` sends work and returns an `mpsc::Receiver<ResponseChunk>`
  - Bridge now carries `stream_tx: mpsc::Sender<ResponseChunk>` + `done_tx: oneshot::Sender<()>` per request
  - New PHP native functions: `folk_write_head(int $status, string $headers_json)`, `folk_write(string $data)`, `folk_write_end()`
  - Backward compat: PHP handlers returning `{status, headers, body}` are auto-converted to chunks

- **folk-plugin-http v0.4.0** — true chunked HTTP responses via `axum::Body::from_stream` ([#35](https://github.com/Folk-Project/folk-releases/issues/35))
  - `handle_inner` switches from `execute_value_traced` to `execute_streamed`; chunks are piped directly to the client when no `response.after` hooks need the body
  - `transfer-encoding: chunked` is sent automatically by hyper for streaming responses

- **folk-builder v0.2.5** — codegens `folk_write_head/folk_write/folk_write_end` in the generated cdylib

- **folk-sdk v0.3.1** — `Folk::writeHead(int $status, array $headers)`, `Folk::write(string $data)`, `Folk::end()`

**PHP usage example:**
```php
// Streaming SSE or large response
\Folk\Sdk\Folk::writeHead(200, ['Content-Type' => 'text/event-stream']);
foreach ($events as $event) {
    \Folk\Sdk\Folk::write("data: {$event}\n\n");
}
\Folk\Sdk\Folk::end();
```

### Versions

| Package | Version | Type |
|---------|---------|------|
| folk-api | 0.2.8 | crates.io |
| folk-core | 0.3.7 | crates.io |
| folk-ext | 0.3.7 | crates.io |
| folk-plugin-http | 0.4.0 | crates.io |
| folk-plugin-grpc | 0.2.6 | crates.io |
| folk-plugin-jobs | 0.3.4 | crates.io |
| folk-plugin-metrics | 0.2.3 | crates.io |
| folk-plugin-process | 0.2.4 | crates.io |
| folk-builder | 0.2.5 | crates.io |
| folk-sdk | 0.3.1 | packagist |
| folk/laravel | 0.3.5 | packagist |
| folk/spiral | 0.1.2 | packagist |
| folk/yii3 | 0.1.1 | packagist |
