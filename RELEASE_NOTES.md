# Release Notes

### gRPC client-streaming & bidi on the server ([#92](https://github.com/Folk-Project/folk-releases/issues/92))

Folk gRPC now supports **all four call modes on both sides**. This release adds the
last missing quadrant: a Folk handler that **receives a stream of request messages**
— client-streaming (N→1) and bidirectional (N↔N). Server-streaming server and all
three streaming clients already shipped in the streaming release ([#32](https://github.com/Folk-Project/folk-releases/issues/32)).

**PHP handlers just `foreach`.** The generated interface types the request side as
`iterable $requests`; you iterate hydrated request DTOs and either return one
response (client-streaming) or `yield` a stream (bidi):

```php
class UploaderService implements UploaderInterface
{
    /** @param iterable<UploadChunk> $requests */
    public function Upload(iterable $requests, Context $context): ?UploadResult
    {
        $bytes = 0;
        foreach ($requests as $chunk) {          // hydrated request DTOs
            $bytes += strlen($chunk->data);
        }
        return new UploadResult(bytes: $bytes);
    }

    /**
     * @param  iterable<ChatMsg> $requests
     * @return iterable<ChatMsg>
     */
    public function Chat(iterable $requests, Context $context): iterable
    {
        foreach ($requests as $msg) {
            yield new ChatMsg(text: "echo: {$msg->text}");
        }
    }
}
```

Registration is identical to any other handler (`folk.grpc.services`). The inbound
element type is carried by the generated interface's `@param iterable<Dto>` docblock
(IDE + phpstan) and an `INPUT_STREAMS` map (the router hydrates each message) — you
never touch a low-level pull primitive.

**Under the hood:** the transport reads the incoming HTTP/2 body one gRPC frame at a
time (never buffering the whole stream), transcodes each to a message, and feeds the
worker through the request-body streaming channel introduced for HTTP — so a PHP
`foreach` pulls one inbound message at a time. Reuses the existing streaming machinery;
**folk-api and folk-builder are unchanged.**

!!! note "v1 bidi is lockstep"
    On a single worker, bidi is **serialized** (the client sends all requests, then
    the responses drain). Fine for request/response echo patterns; truly concurrent
    bidi is a later increment. Client-streaming/bidi are **transcode-mode only** (the
    server needs the descriptor to detect the call kind). See
    [gRPC → Streaming](https://folk-project.github.io/folk-releases/plugins/grpc/#streaming).

Verified end-to-end with real Folk on both ends across **all four framework stands**
(Laravel, Symfony, Spiral, Yii): client-streaming aggregation, bidi echo, empty
streams, plus server-streaming + unary negative controls.

**Versions:** folk-core/folk-ext **0.6.3** (`folk_grpc_recv` inbound host fn),
folk-plugin-grpc **0.7.1** (client-streaming/bidi server dispatch), folk/sdk **0.4.7**
(inbound router path, generated `iterable`-request interfaces). folk-api (0.3.3),
folk-builder (0.2.20), and the framework adapters are unchanged (they resolve
folk/sdk `^0.4`). Prebuilt extension release `0.2.13`.
