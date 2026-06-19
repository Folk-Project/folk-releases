### What's new

- **folk/laravel v0.3.4, folk/symfony v0.1.1** — request_id in application logs ([#38](https://github.com/Folk-Project/folk-releases/issues/38))
  - Application logs now carry the current `request_id` automatically, so `Log::info(...)` (Laravel) and `$logger->info(...)` (Symfony) records can be correlated with the id returned by `\Folk\Sdk\Folk::requestId()`
  - Implemented as a Monolog processor that reads the id **at log time** — stateless, so it never leaks a stale id between requests on a recycled worker, and adds nothing when there is no request in flight (id `0`)
  - **Laravel**: attached to the default log channel automatically
  - **Symfony**: attached when the logger is Monolog (e.g. `symfony/monolog-bundle` installed); a no-op otherwise, so apps without Monolog are unaffected
  - Requires `folk/sdk ^0.2.7`. Spiral and Yii3 adapters will follow in a later release

- **folk-core / folk-ext v0.2.10, folk-sdk v0.2.7, folk-builder v0.2.3** — Request IDs ([#34](https://github.com/Folk-Project/folk-releases/issues/34))
  - Every request now carries a unique, monotonic `request_id` threaded end-to-end from the Rust dispatcher into the PHP worker
  - New native function `folk_request_id()` and SDK facade `\Folk\Sdk\Folk::requestId()` return the current request's id (0 outside a request) — use it to correlate PHP application logs with Folk's Rust-side access logs
  - New `[workers]` setting `max_concurrent_per_worker` (default `1`). Only `1` is supported today; values `> 1` are reserved for a future async runtime (PHP Fibers) and are clamped to `1` with a warning. No behavior change for existing setups
  - Foundation for upcoming response streaming ([#35](https://github.com/Folk-Project/folk-releases/issues/35))

- **folk-plugin-process v0.2.3** — Empty/absent `[process]` config no longer crashes ([#36](https://github.com/Folk-Project/folk-releases/issues/36))
  - A config with no `[process]` section (or an empty one) previously failed on startup with `Config error: invalid [process] configuration`. The process plugin is always compiled in, so an absent section is a normal "supervise nothing" case — it now starts cleanly, like the other plugins
  - Config errors now surface the underlying reason (e.g. `missing field \`command\``) instead of a generic message

- **folk-core / folk-ext v0.2.9** — Hot reload / dev watch mode ([#28](https://github.com/Folk-Project/folk-releases/issues/28))
  - New `[dev]` config: `watch`, `watch_paths`, `watch_extensions`, `debounce`. Disabled by default
  - A file watcher recycles workers when watched `.php` files change — code edits are picked up without a manual restart; in-flight requests complete first (graceful)
  - Requires PHP ZTS with `workers.count > 1`: the main PHP thread (worker #1) is not recyclable, so single-worker servers cannot fully hot reload
  - No OPcache reset on reload (it raced live worker recycling and could crash). Keep `opcache.validate_timestamps = 1` in dev (default) so edits are seen
  - Also: forward-compat with `ext-php-rs` 0.15.15 (`ArrayKey::ZendString`)

### Versions

| Package | Version | Type |
|---------|---------|------|
| folk-core | 0.2.10 | crates.io |
| folk-ext | 0.2.10 | crates.io |
| folk-sdk | 0.2.7 | packagist |
| folk-builder | 0.2.3 | crates.io |
| folk-plugin-process | 0.2.3 | crates.io |
| folk/laravel | 0.3.4 | packagist |
| folk/symfony | 0.1.1 | packagist |
