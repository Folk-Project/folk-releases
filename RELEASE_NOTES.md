# Release Notes

### gRPC client — call upstream services from PHP, no protoc ([#88](https://github.com/Folk-Project/folk-releases/issues/88))

**Call external gRPC services from PHP with typed DTOs — the client counterpart
to #87, still no `protoc` / `ext-grpc`.**

Declare a named upstream and point it at its `.proto`; the transport (address,
TLS, deadline, retries) stays in `folk.toml`, PHP never sees an endpoint:

```toml
[grpc.clients.catalog]
proto = ["proto/clients/catalog.proto"]
address = "catalog.svc:50051"   # or a list → round-robin load balancing
deadline = "5s"
```

Generate a typed client stub (`--client` on the same generator) and call it —
DTO in, DTO out:

```php
$catalog = Folk::grpcClient(CatalogClient::class);
$resp = $catalog->Search(new SearchRequest(query: 'phone'));   // DTO → DTO
foreach ($resp->products as $p) { echo $p->title; }
```

**Highlights**

- The phase-87 transcoding runs **in reverse** — Rust encodes the request and
  decodes the response against the upstream's descriptor pool. Each
  `[grpc.clients.<name>]` has its **own** proto/pool (upstreams may reuse message
  names without colliding).
- Full type map on request *and* response (repeated/map/enum/bytes/oneof/
  well-known), shared with the server DTOs.
- Per-call **deadline** (`->withDeadline()`, propagated as `grpc-timeout`; expiry →
  `DEADLINE_EXCEEDED(4)`), outbound **metadata** (`->withMetadata()`), **retries**
  on transient `UNAVAILABLE` only (never a business status), and client-side
  **load balancing** across multiple addresses.
- Upstream **TLS/mTLS**, distinct from the server listener.
- Errors surface as `Folk\Sdk\Grpc\GrpcException` (`$e->status()`): business status
  passed through; unreachable upstream → `UNAVAILABLE(14)`.
- **Client-only deployments**: omit `[grpc] listen` to make outbound calls without
  binding a server.
- **Synchronous** call model (worker blocks on the RPC bridge, like `jobs.push`) —
  always set a deadline; channels are pooled per worker.

`folk-api` and `folk-builder` are unchanged. Unary only; streaming is tracked in
[#32](https://github.com/Folk-Project/folk-releases/issues/32).

Validated end-to-end on **all four** smoke stands (Laravel, Symfony, Spiral, Yii3):
DTO round-trip across the full type map, business status → exception, unreachable
upstream → `UNAVAILABLE`, per-call deadline, outbound metadata, and client-only
mode — each calling a shared upstream Folk gRPC server.

Ships in **folk-plugin-grpc 0.5.0** (crates.io), **folk/sdk** and **folk/laravel**
(Packagist), prebuilt `.so` build-manifest bumped.

---

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
