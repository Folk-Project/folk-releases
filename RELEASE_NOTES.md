# Release Notes

### gRPC proto DX — transcoding + DTO generation, no protoc ([#87](https://github.com/Folk-Project/folk-releases/issues/87))

**Work with protobuf from PHP using typed DTOs — on the request *and* the
response — without `protoc` or `ext-protobuf`.**

Set `[grpc] transcode = true` and Folk decodes each incoming protobuf message to
a native value **in Rust** (via the proto descriptor) and re-encodes the response
the same way. Handlers then work with generated, plain readonly DTOs instead of
raw bytes:

```php
final class GreeterService implements GreeterInterface
{
    public function SayHello(HelloRequest $request, Context $context): ?HelloReply
    {
        return new HelloReply(message: "Hello, {$request->name}!");
    }
}
```

**Generate the DTOs, enums, and `*Interface` contracts from your `.proto`** — no
protoc in the chain (the gRPC plugin compiles the descriptor set itself):

- Laravel: `php artisan folk:grpc:generate`
- Any framework: `vendor/bin/folk-grpc-gen --out app/Grpc/Generated --namespace 'App\Grpc\Generated' proto/*.proto`

**Highlights**

- Full type map: scalars, `repeated`→`list`, `map`, `enum` (int-backed, unknown→zero),
  `bytes` (base64 on the wire, raw in the DTO), `oneof`, and well-known types
  (Timestamp/Duration/FieldMask→string, wrappers→nullable, Struct/Value/ListValue→array,
  `Any`→array with typed unpacking + base64 fallback).
- Two tiers, chosen by the wire: generated DTOs (transcode) or the legacy raw
  passthrough (`transcode = false`, default — **zero regression**).
- Richer `Context`: metadata multimap (`getValue`/`getAll`), `bearerToken`,
  `getBinary`, `service`/`method`, `requestId` (UUID v7), advisory deadline
  (`remaining()`), peer/authority.
- Descriptor pool from files, **directories**, or a prebuilt `descriptor_set`
  bundle; gRPC reflection and health unchanged.

`folk-api` is unchanged. **Calling** external gRPC services from PHP (the client
side) lands in a follow-up.

Validated end-to-end on the Laravel and Spiral smoke stands (grpcurl round-trips
across the full type map, business status codes, Context, and a `transcode = false`
passthrough negative control).

Ships in **folk-plugin-grpc 0.4.0**, **folk-builder 0.2.17** (crates.io),
**folk/sdk 0.4.3**, **folk/laravel 0.4.4** (Packagist), prebuilt `.so` **0.2.8**.
