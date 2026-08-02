# Release Notes

### gRPC DTOs: get/set accessors

The generated gRPC message DTOs now use a familiar **get/set** shape instead of
public readonly properties. This is a **PHP-only** change in `folk/sdk` — no Rust,
no `.so` rebuild, no config change.

**What changed.** A generated message used to be a `final readonly class` with public
promoted constructor properties — you could only build it through the constructor and
read fields directly (`$dto->name`). Now each message is a `final class` with `private`
fields plus fluent accessors:

```php
// build — both work, and are equivalent on the wire
$reply = (new HelloReply())->setMessage("Hi");   // fluent, chainable
$reply = new HelloReply(message: "Hi");          // named args — constructor kept

// read
echo $reply->getMessage();
```

Every field gets `getX(): T` and `setX(T $value): self`. The all-optional constructor
is preserved, so existing `new Foo(field: ...)` construction keeps working unchanged.

**Protected behavior — unchanged and verified.** The wire contract (canonical JSON:
snake_case fields, enums as ints, bytes base64) is byte-identical; `FOLK_FIELDS`, the
`Hydrator::hydrate()` build path, service `*Interface` and `*Client` stub signatures,
`INPUT_STREAMS`, the `{package}` layout, and int-backed enums are all untouched.
Internally `Hydrator::dehydrate()` now reads the private fields by reflection on the
field name (decoupled from accessor naming). The full folk/sdk suite (110 tests,
phpstan level 8) is green, and end-to-end gRPC — unary, server-streaming,
client-streaming, and bidi, plus the full type-map `Echo` — was smoke-tested across
Laravel, Symfony, Spiral, and Yii 3.

**Upgrading (breaking for regenerated code).** After upgrading, **re-run codegen**
(`php artisan folk:grpc:generate`, or `vendor/bin/folk-grpc-gen`) and switch field
**reads** from `$dto->field` to `$dto->getField()`. Construction via
`new Foo(field: ...)` needs no change. Because the fields are now private,
`json_encode($dto)` no longer emits them — go through the SDK path (the router and
client already do) or build the array from the getters.

**Versions:** folk/sdk **0.4.8** (Packagist). No other package changed — the framework
adapters resolve `folk/sdk` on `^0.4` and need no update; folk-api, folk-core/folk-ext,
the plugins, folk-builder, and the prebuilt extension are all untouched.
