### What's new

- **folk-core / folk-ext v0.3.0, folk-sdk v0.3.0, folk-api v0.2.2, folk-builder v0.2.4, folk-plugin-http v0.2.5, folk/laravel v0.3.5, folk/symfony v0.1.2** — Globally-unique `request_id` (UUID) ([#48](https://github.com/Folk-Project/folk-releases/issues/48))
  - `request_id` is now a globally-unique **UUID v7** (time-ordered, so it still sorts by creation time) instead of a per-process counter. It no longer collides across instances or restarts, so it works as a single correlation key in aggregated logs (Loki, ELK, Grafana) without also filtering by host/pod
  - The id now also appears in the **HTTP access log** (`request_id` field, when `access_log = true`) — so Folk's Rust-side access line and your PHP application log of the same request share one id
  - ⚠️ **Breaking:** `\Folk\Sdk\Folk::requestId()` and the native `folk_request_id()` now return a **`string`** (the UUID, or `""` outside a request) instead of `int`. Update any code that treated the id as an integer. The folk/laravel and folk/symfony log processors handle this automatically
  - Adapters require `folk/sdk ^0.3.0`

- **folk/laravel v0.3.4, folk/symfony v0.1.1** — request_id in application logs ([#38](https://github.com/Folk-Project/folk-releases/issues/38))
  - Application logs now carry the current `request_id` automatically, so `Log::info(...)` (Laravel) and `$logger->info(...)` (Symfony) records can be correlated with the id returned by `\Folk\Sdk\Folk::requestId()`
  - Implemented as a Monolog processor that reads the id **at log time** — stateless, so it never leaks a stale id between requests on a recycled worker, and adds nothing when there is no request in flight (id `0`)
  - **Laravel**: attached to the default log channel automatically
  - **Symfony**: attached when the logger is Monolog (e.g. `symfony/monolog-bundle` installed); a no-op otherwise, so apps without Monolog are unaffected
  - Requires `folk/sdk ^0.2.7`. Spiral and Yii3 adapters will follow in a later release

### Versions

| Package | Version | Type |
|---------|---------|------|
| folk-core | 0.3.0 | crates.io |
| folk-ext | 0.3.0 | crates.io |
| folk-api | 0.2.2 | crates.io |
| folk-sdk | 0.3.0 | packagist |
| folk-builder | 0.2.4 | crates.io |
| folk-plugin-http | 0.2.5 | crates.io |
| folk-plugin-process | 0.2.3 | crates.io |
| folk/laravel | 0.3.5 | packagist |
| folk/symfony | 0.1.2 | packagist |
