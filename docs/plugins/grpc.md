# gRPC Plugin

Native gRPC server with reflection and health checking. Built on [tonic](https://github.com/hyperium/tonic).

## Features

- Unary gRPC calls with dispatch to PHP workers
- **Typed proto DX**: proto↔native transcoding + DTO/interface generation, no protoc ([see below](#typed-proto-dx-no-protoc))
- **gRPC client**: call upstream services from PHP with typed DTOs, no protoc — deadlines, retries, load balancing, TLS/mTLS ([see below](#grpc-client--call-upstream-services-no-protoc))
- **Streaming**: all four modes on both sides — server/client/bidi handlers (`yield` / `iterable $requests`) and server/client/bidi clients (`foreach`/iterable), typed DTOs, deadlines, cancellation ([see below](#streaming))
- Server reflection via proto files (grpcurl, Postman auto-discovery)
- Automatic proto import resolution
- gRPC Health Checking Protocol (grpc.health.v1)
- TLS/SSL via rustls
- HTTP/2 keepalive
- Max message size limits (human-readable: `"4mb"`)
- Server-wide RPC timeout
- Max concurrent streams
- gRPC compression (gzip)
- Metadata passthrough to PHP
- 6 Prometheus metrics

## Planned

- Truly concurrent bidi on a single worker (v1 bidi is lockstep — all requests sent, then responses drained; [see below](#worker-blocking-model-important)).

## Configuration

```toml
[grpc]
listen = "0.0.0.0:50051"                  # Listening address
proto = ["proto/greeter.proto"]            # Proto files/dirs for reflection + transcoding
transcode = false                          # Typed DTOs instead of raw bytes (see "Typed proto DX")
# descriptor_set = "all.pb"                # Optional prebuilt FileDescriptorSet, merged into the pool
max_recv_message_size = "4mb"              # Max incoming message size
max_send_message_size = "4mb"              # Max outgoing message size
timeout = "30s"                            # Server-wide RPC timeout
max_concurrent_streams = 200               # HTTP/2 concurrent streams limit
compression = false                        # Enable gzip compression

# HTTP/2 keepalive
# [grpc.keepalive]
# interval = "60s"                         # PING interval
# timeout = "20s"                          # PING timeout before disconnect

# TLS — if set, the server listens on secure gRPC
# [grpc.tls]
# cert = "/path/to/cert.pem"
# key = "/path/to/key.pem"
```

When `proto` is non-empty, gRPC server reflection is enabled — tools like `grpcurl` and Postman can discover services automatically. The gRPC Health Checking Protocol is always enabled.

### Configuration reference

| Option | Type | Default | Description |
|--------|------|---------|-------------|
| `listen` | string | — | Server listen address. No default — omit it for a client-only deployment (no server bound). |
| `proto` | array | `[]` | Proto files **or directories** for reflection + transcoding |
| `transcode` | bool | `false` | Decode proto↔native so handlers use typed DTOs (see below) |
| `descriptor_set` | string | — | Prebuilt `FileDescriptorSet` bundle merged into the pool |
| `max_recv_message_size` | size | `"4mb"` | Max incoming message |
| `max_send_message_size` | size | `"4mb"` | Max outgoing message |
| `timeout` | duration | — | Server-wide RPC timeout |
| `max_concurrent_streams` | int | — | HTTP/2 stream limit |
| `compression` | bool | `false` | Enable gzip compression |

### Keepalive

```toml
[grpc.keepalive]
interval = "60s"     # Send HTTP/2 PING frames at this interval
timeout = "20s"      # Close connection if PONG not received
```

Keepalive is off unless the section is present; when it is, both `interval` and `timeout` are required (they have no defaults).

### TLS

```toml
[grpc.tls]
cert = "/etc/ssl/certs/server.crt"
key = "/etc/ssl/private/server.key"
```

Uses rustls (no OpenSSL dependency). When TLS is configured, the server automatically supports HTTP/2 via ALPN.

### Compression

When `compression = true`, the server:

- Decompresses incoming messages with gzip (`grpc-encoding: gzip`)
- Compresses outgoing messages if client sends `grpc-accept-encoding: gzip`
- Sets `grpc-encoding: gzip` response header when compressing

## Health Checking

The gRPC Health Checking Protocol (`grpc.health.v1.Health`) is always enabled:

```bash
grpcurl -plaintext localhost:50051 grpc.health.v1.Health/Check
# {"status": "SERVING"}
```

## Typed proto DX (no protoc)

With `transcode = true`, Folk decodes each incoming protobuf message to a native
value **in Rust** (via the proto descriptor) and re-encodes the response the same
way. PHP handlers then work with plain, typed **DTOs** instead of raw bytes — and
Folk generates those DTOs for you. **No `protoc`, no `ext-protobuf`.**

### 1. Enable transcoding

```toml
[grpc]
proto = ["proto"]        # a file or a directory (scanned recursively for *.proto)
transcode = true
```

`proto` accepts files **and directories**; an optional `descriptor_set = "all.pb"`
merges a prebuilt `FileDescriptorSet` into the pool (useful so `google.protobuf.Any`
can unpack extra types). Leaving `transcode = false` keeps the raw passthrough
behaviour unchanged.

### 2. Generate DTOs and interfaces

The generator compiles your `.proto` with the gRPC plugin (in-process when the
`folk` extension is loaded, otherwise via a short-lived `folk-server
grpc:descriptors` subprocess) and writes DTOs (private fields with `get*`/`set*`
accessors), int-backed enums, and a `*Interface` service contract.

- **Laravel:**

  ```bash
  php artisan folk:grpc:generate
  ```

  Configure in `config/folk.php` — a `server` block (contracts you implement) and
  a `clients` block (stubs you call); `services` maps a service to its handler:

  ```php
  'grpc' => [
      'services' => [
          \App\Grpc\Generated\Server\Io\Altessa\Serviceinfo\V1\ServiceInfoServiceInterface::class
              => \App\Grpc\ServiceInfoService::class,
      ],

      'server' => [
          // List entry-point *service* .proto files; imports are compiled
          // automatically. Point at files, not a directory (a directory sweeps
          // imported protos into the compile set and double-loads them).
          'proto' => ['proto/io/altessa/serviceinfo/v1/serviceinfo_service.proto'],
          // Defaults (omit to use them):
          'generated_dir'       => null,  // app_path('Grpc/Generated/Server')
          'generated_namespace' => null,  // 'App\Grpc\Generated\Server\{package}'
      ],

      'clients' => [
          'catalog' => [
              'proto' => ['proto/clients/catalog.proto'],
              // Defaults: app_path('Grpc/Generated/Client/{client_name}')
              //           'App\Grpc\Generated\Client\{client_name}\{package}'
              'generated_dir' => null,
              'generated_namespace' => null,
          ],
      ],
  ],
  ```

- **Symfony, Spiral, Yii3** (and any framework) — the shared CLI shipped by
  `folk/sdk` (same placeholders):

  ```bash
  vendor/bin/folk-grpc-gen --out app/Grpc/Generated/Server \
      --namespace 'App\Grpc\Generated\Server\{package}' \
      proto/io/altessa/serviceinfo/v1/serviceinfo_service.proto
  ```

#### Layout: `{package}`, `{service_name}`, `{client_name}`

The `generated_namespace` (and `generated_dir`) may contain placeholders:

- **`{package}`** — nests each class by its proto package, e.g.
  `io.altessa.serviceinfo.v1` → `Io\Altessa\Serviceinfo\V1`. This is the default,
  and mirrors `protoc-gen-php`. Cross-package field types reference each other by
  fully-qualified name; same-package types by short name. Omit `{package}` for a
  flat layout (everything in one namespace).
- **`{client_name}`** — the `clients.<name>` key (client role only).

So the default server tree looks like:

```
app/Grpc/Generated/Server/
├── Io/Altessa/Serviceinfo/V1/{ServiceInfo, ServiceInfoServiceInterface}.php
└── Io/Altessa/Type/V1/FileRef.php          # referenced by ServiceInfo via FQN
```

Server and client trees are independent — a message used by both is generated in
each (no shared tree), so a server and a client that vendor different versions of
the same proto never collide.

> The top-level `grpc.proto` / `generated_dir` / `generated_namespace` keys also
> work and produce the flat layout; put them under `grpc.server` for the package
> layout.

Generation is idempotent: it overwrites the target directory and stamps every
file with `// @generated by Folk — do not edit`.

### 3. Implement and register

The generated `GreeterInterface` carries the service name and typed signatures:

```php
use App\Grpc\Generated\GreeterInterface;
use App\Grpc\Generated\HelloRequest;
use App\Grpc\Generated\HelloReply;
use Folk\Sdk\Grpc\Context;

final class GreeterService implements GreeterInterface
{
    public function SayHello(HelloRequest $request, Context $context): HelloReply
    {
        return new HelloReply(message: "Hello, {$request->getName()}!");
    }
}
```

Each message DTO has `private` fields, a `get*()` per field, and a fluent
`set*($value): self`. Build them either through the all-optional constructor
(named arguments) or by chaining setters — both are equivalent on the wire:

```php
$reply = (new HelloReply())->setMessage("Hi");     // fluent
$reply = new HelloReply(message: "Hi");            // named args — same result
```

> DTOs are **mutable** (not `readonly`). Read fields with the getters, not direct
> property access. Because the fields are private, `json_encode($dto)` will not
> emit them — go through the SDK path (the router/client already do) or build the
> array yourself from the getters.

Register it (service name comes from the interface's `NAME` constant):

```php
'grpc' => [
    'services' => [
        \App\Grpc\Generated\GreeterInterface::class => \App\Grpc\GreeterService::class,
    ],
],
```

### Two tiers

The router dispatches on the wire envelope, so both models coexist:

| Tier | When | Handler signature | Notes |
|------|------|-------------------|-------|
| **Transcode (DTO)** | `transcode = true` | `method(RequestDto $r, Context $c): ReplyDto` | Recommended. Typed in and out. |
| **Passthrough** | `transcode = false` | `method(string $bytes): string` or a `protoc`-generated `Message` | Lower-level. Raw protobuf bytes. |

### Type map

| proto | PHP |
|-------|-----|
| `double`/`float` | `float` |
| `int32/64`, `uint32/64`, `sint*`, `fixed*` | `int` (uint64 > 2^63 documented edge) |
| `bool` | `bool` |
| `string` | `string` |
| `bytes` | `string` (base64 on the wire, **raw** bytes in the DTO) |
| `enum` | int-backed PHP `enum` (`tryFrom`, unknown value → zero-value case) |
| `message` | nested DTO (`?Dto`, null when absent) |
| `repeated T` | `list<T>` |
| `map<K,V>` | `array<K,V>` |
| `oneof` | nullable members; only the active branch is set |
| `Timestamp`/`Duration`/`FieldMask` | `string` (canonical JSON) |
| wrappers (`*Value`) | nullable scalar |
| `Struct`/`Value`/`ListValue` | `array`/`mixed`/`list<mixed>` |
| `Any` | `array` (`@type` + fields; known types unpacked, unknown → base64 fallback) |

### Context (call metadata)

The `Context` passed to every handler exposes request metadata and the deadline:

| Method | Returns |
|--------|---------|
| `getValue($k)` / `getAll($k)` | first value / all values (HTTP/2 repeats keys) |
| `has($k)` | key present (case-insensitive) |
| `getBinary($k)` | base64-decoded `-bin` metadata |
| `authorization()` / `bearerToken()` | `Authorization` header / its bearer token |
| `service()` / `method()` | fully-qualified service / method name |
| `requestId()` | correlation id (UUID v7), also on the Rust logs |
| `hasDeadline()` / `timeoutSeconds()` / `remaining()` | client `grpc-timeout` (advisory — Folk can't force-kill a blocking worker) |
| `peerAddress()` / `authority()` | remote peer (proxy-aware, like XFF on HTTP) / `:authority` |

## PHP Handler (passthrough)

Without transcoding, register a handler that works with raw protobuf bytes (or a
`protoc`-generated `Message`), via the SDK's `GrpcRouter`:

```php
$router = new \Folk\Sdk\Grpc\GrpcRouter();
$router->register('greeter.Greeter', new GreeterService());

$loop = new \Folk\Sdk\Worker\WorkerLoop();
$loop->registerGrpcHandler($router);
$loop->run();
```

In the framework adapters this wiring is automatic — you only list services under
`folk.grpc.services` (see above).

## Status codes

Return a successful response as usual. To report a **business outcome** (the
standard server-side gRPC idiom), call `$context->setStatus($code, $message)`
and return `null` — the plugin maps it to the gRPC status code:

```php
use Folk\Sdk\Grpc\Context;

public function GetUser(GetUserRequest $request, Context $context): ?UserReply
{
    $user = $this->repo->find($request->getId());
    if ($user === null) {
        $context->setStatus(5, 'user not found');   // NOT_FOUND
        return null;
    }
    return new UserReply(name: $user->name);
}
```

`$code` is a canonical [`google.rpc.Code`](https://grpc.io/docs/guides/status-codes/):

| code | name |
|------|------|
| 3 | INVALID_ARGUMENT |
| 5 | NOT_FOUND |
| 6 | ALREADY_EXISTS |
| 7 | PERMISSION_DENIED |
| 8 | RESOURCE_EXHAUSTED |
| 12 | UNIMPLEMENTED |
| 13 | INTERNAL |
| 14 | UNAVAILABLE |
| 16 | UNAUTHENTICATED |

An **uncaught exception** in a handler is a fatal error: the call fails with
`INTERNAL (13)`. The exception class and stack trace are included in the status
message only in dev mode (`[dev] watch`); in production the client gets a
generic message and the full detail is logged server-side.

## gRPC client — call upstream services (no protoc)

Folk can also **call** external gRPC services, with the same typed DTO-in /
DTO-out ergonomics and no `protoc` / `ext-grpc`. The transcoding machinery runs
in reverse: PHP hands Folk a request DTO, Rust encodes it against the upstream's
descriptor, makes the unary call, and decodes the response back into a DTO.

### 1. Declare the upstream

Each `[grpc.clients.<name>]` is a named upstream with its **own** proto contract
(its own descriptor pool — two upstreams may reuse message names without
colliding). The transport (address, TLS, deadline, retries) lives here; PHP never
sees an endpoint.

```toml
[grpc.clients.catalog]
proto = ["proto/clients/catalog.proto"]
address = "catalog.svc:50051"   # or ["a:50051", "b:50051"] → round-robin load balancing
deadline = "5s"                 # default per-call deadline
# descriptor_set = "catalog.pb" # optional prebuilt FileDescriptorSet merged into this client's pool

[grpc.clients.catalog.retries]  # transient (UNAVAILABLE) only — never a business status
max_attempts = 3
backoff = "100ms"

[grpc.clients.catalog.tls]      # optional upstream TLS/mTLS
ca = "/etc/ssl/ca.pem"
# cert = "/etc/ssl/client.pem"  # + key for mTLS
# domain = "catalog.internal"   # SNI override
```

No `[grpc] listen`? That's fine — a config with only `[grpc.clients.*]` is a
valid **client-only** deployment (Folk makes outbound calls, binds no server).

### 2. Generate the client stub

Same generator as the server side, `--client` flag selects the stub:

- Laravel: add the upstream under `folk.grpc.clients` in `config/folk.php`, then
  `php artisan folk:grpc:generate` (emits server contracts **and** every client
  stub), or `--client=catalog` for just one.
- Any framework:
  `vendor/bin/folk-grpc-gen --client catalog --out app/Grpc/Clients/Catalog --namespace 'App\Grpc\Clients\Catalog' proto/clients/catalog.proto`

This emits a `CatalogClient` (one typed method per unary RPC) plus its DTOs/enums.

### 3. Call it

```php
use Folk\Sdk\Folk;
use Folk\Sdk\Grpc\GrpcException;

$catalog = Folk::grpcClient(CatalogClient::class);            // resolves [grpc.clients.catalog]
// or Folk::grpcClient(CatalogClient::class, 'other:50051');  // endpoint override

$resp = $catalog->Search(new SearchRequest(query: 'phone', page: 1));  // DTO in → DTO out
foreach ($resp->getProducts() as $p) {
    echo $p->getTitle();
}
```

Per-call metadata and deadline are fluent (immutable — each returns a new client):

```php
$catalog
    ->withMetadata('authorization', "Bearer {$token}")
    ->withDeadline(0.5)
    ->Search($req);
```

A non-OK result throws `Folk\Sdk\Grpc\GrpcException` (`$e->status()` is a canonical
gRPC code): a business status the upstream set (e.g. `NOT_FOUND(5)`), an expired
deadline (`DEADLINE_EXCEEDED(4)`), or an unreachable upstream (`UNAVAILABLE(14)`).

### Worker-blocking model

The call is **synchronous**: the PHP worker blocks on the RPC bridge until the
upstream responds (the same competing-consumers model as `jobs.push`). A slow
upstream ties up a worker, so **always set a deadline** and size your worker pool
accordingly. Channels are pooled per worker (lazy, kept alive across requests).

## Streaming

Folk streams gRPC with the same no-`protoc` typed-DTO model, over a dedicated
async bridge (the unary path is strictly one-in-one-out). A streaming RPC is
declared in `.proto` the usual way; the generator emits the streaming shapes.

**All four gRPC modes work on both sides:**

- **server-streaming server** — a Folk handler *yields* a stream of responses;
- **client-streaming / bidi server** — a Folk handler *receives* a request stream
  via `iterable $requests` (a plain `foreach`);
- **server / client / bidirectional streaming client** — PHP drains or feeds a
  stream over a `foreach` / iterable.

### Server-streaming handler (server)

The generated interface types a server-streaming method as `iterable`; the
handler `yield`s response DTOs, one gRPC message per `yield`:

```php
use Folk\Sdk\Grpc\Context;

class PricesService implements PricesInterface
{
    /** @return iterable<PriceUpdate> */
    public function Watch(WatchRequest $request, Context $context): iterable
    {
        foreach ($this->feed($request->getTopic()) as $tick) {
            yield new PriceUpdate(symbol: $tick->symbol, price: $tick->price);
        }
        // A business status set here (or mid-stream) ends the stream with that code:
        // $context->setStatus(9, 'feed closed');
    }
}
```

Registration is identical to a unary handler (`folk.grpc.services`). A `return`
(no `yield`) is a valid empty stream.

### Client-streaming / bidi handler (server)

When the request side is a stream, the generated interface types the method's
first parameter as `iterable $requests` — the handler `foreach`es the inbound
request DTOs. **Client-streaming** returns one response DTO; **bidi** `yield`s a
stream of responses while consuming the requests:

```php
use Folk\Sdk\Grpc\Context;

class UploaderService implements UploaderInterface
{
    // client-streaming: a stream of requests → one response
    /** @param iterable<UploadChunk> $requests */
    public function Upload(iterable $requests, Context $context): ?UploadResult
    {
        $bytes = 0;
        foreach ($requests as $chunk) {          // each is a hydrated request DTO
            $bytes += strlen($chunk->getData());
        }
        return new UploadResult(bytes: $bytes);
    }

    // bidi: a stream of requests ↔ a stream of responses (v1 lockstep — see below)
    /**
     * @param  iterable<ChatMsg> $requests
     * @return iterable<ChatMsg>
     */
    public function Chat(iterable $requests, Context $context): iterable
    {
        foreach ($requests as $msg) {
            yield new ChatMsg(text: "echo: {$msg->getText()}");
        }
    }
}
```

The element type of `iterable $requests` lives in the generated interface's
`@param iterable<Dto>` docblock (so IDEs and phpstan see it) and its
`INPUT_STREAMS` map (so the router hydrates each message) — you never touch the
low-level pull primitive. As with unary, `$context->setStatus()` reports a
business status; for client-streaming, return `null` alongside it.

### Streaming client

The generated `{Service}Client` extends `GrpcStreamClient` and exposes one typed
method per streaming RPC. Construct it with `Folk::grpcClient()` exactly like the
unary client; `withMetadata()` / `withDeadline()` apply.

```php
$prices = Folk::grpcClient(PricesClient::class);

// server-streaming: one request → a lazy stream of responses
foreach ($prices->Watch(new WatchRequest(topic: 'FScoin')) as $update) {
    echo "{$update->getSymbol()} {$update->getPrice()}\n";   // arrives one message at a time
}

// client-streaming: a stream of requests → one response
$summary = $uploader->Upload((function () {
    foreach ($chunks as $c) {
        yield new UploadChunk(data: $c);
    }
})());

// bidirectional: a stream of requests ↔ a stream of responses
foreach ($chat->Converse($outgoing) as $incoming) {
    echo $incoming->getText();
}
```

A non-OK trailing status (business status, or a transport failure) throws
`GrpcException` out of the `foreach`, after any messages already delivered.

### Deadlines and cancellation

- **Deadline** — `withDeadline($s)` (or the client's default) bounds the **whole**
  stream. If it elapses before the stream finishes, the `foreach` throws
  `DEADLINE_EXCEEDED(4)`. A stream with no deadline runs until it completes or the
  consumer stops.
- **Cancellation** — `break` out of the `foreach` (or let the generator go out of
  scope) and Folk cancels the RPC upstream.

### Worker-blocking model (important)

A stream **holds its worker for its entire lifetime**: while PHP is parked in
`foreach`/`recv`, that worker handles nothing else (the competing-consumers model,
same as unary but for the stream's whole duration). Long-lived streams can exhaust
a small pool — **set deadlines** and size `[grpc] workers` (or `[workers] count`)
for the number of concurrent streams you expect. Backpressure is built in: a slow
consumer blocks on a bounded channel rather than buffering the stream in memory.

> v1 bidi on a single worker is **serialized** (all requests are sent, then
> responses are drained) — true concurrent bidi is a later increment.

## Testing

```bash
# List services (reflection)
grpcurl -plaintext localhost:50051 list

# Call a method
grpcurl -plaintext -d '{"name": "World"}' \
    localhost:50051 greeter.Greeter/SayHello

# Health check
grpcurl -plaintext localhost:50051 grpc.health.v1.Health/Check
```

## Metrics

The gRPC plugin registers the following Prometheus metrics (via the metrics plugin):

| Metric | Type | Labels | Description |
|--------|------|--------|-------------|
| `folk_grpc_requests_total` | Counter | `service`, `method`, `grpc_status` | Total gRPC calls |
| `folk_grpc_request_duration_seconds` | Histogram | `service`, `method` | Processing time |
| `folk_grpc_message_sent_bytes` | Histogram | `service`, `method` | Outgoing message size |
| `folk_grpc_message_received_bytes` | Histogram | `service`, `method` | Incoming message size |
| `folk_grpc_active_streams` | Gauge | — | In-flight gRPC calls (incremented per request, including unary) |
| `folk_grpc_errors_total` | Counter | `service`, `method`, `type` | Errors by type |
