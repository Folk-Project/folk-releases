# Release Notes

### gRPC streaming — server-streaming handlers and a streaming client, no protoc ([#32](https://github.com/Folk-Project/folk-releases/issues/32))

**Stream gRPC from PHP with the same typed-DTO model — a handler that `yield`s a
stream of responses, and a client that drains or feeds a stream over `foreach`.**

Streaming runs over a new async PHP↔Rust bridge (the unary path is strictly
one-in-one-out). A streaming RPC is declared in `.proto` as usual; the generator
emits the streaming shapes.

Server — a **server-streaming** handler yields response DTOs, one gRPC message
per `yield`:

```php
/** @return iterable<PriceUpdate> */
public function Watch(WatchRequest $request, Context $context): iterable
{
    foreach ($this->feed($request->topic) as $tick) {
        yield new PriceUpdate(symbol: $tick->symbol, price: $tick->price);
    }
}
```

Client — the generated stub extends `GrpcStreamClient`; drain a server-stream
with `foreach`, feed a client-stream with an iterable, or both for bidi:

```php
foreach ($prices->Watch(new WatchRequest(topic: 'FScoin')) as $update) {
    echo "{$update->symbol} {$update->price}\n";     // one message at a time
}
```

**Highlights**

- **server-streaming server** + **server / client / bidirectional streaming
  client** — all with typed DTOs, no `protoc` / `ext-grpc`. (Client-streaming and
  bidi on the *server* side are a later increment; the client does all three.)
- A per-call **deadline** bounds the whole stream → `DEADLINE_EXCEEDED(4)`; a
  trailing business status throws `GrpcException` out of the `foreach`; `break`
  cancels the RPC upstream.
- **Worker-blocking model**: a stream holds its worker for its whole lifetime
  (competing-consumers) — set deadlines and size the pool for concurrent streams.
  Backpressure is built in (bounded channels, no whole-stream buffering).
- Reuses the phase-87/88 config verbatim — no new `folk.toml` keys.

Requires the streaming host functions in the extension (rebuild the prebuilt
`.so` / `folk-builder`). See [gRPC streaming](docs/plugins/grpc.md#streaming-32).
