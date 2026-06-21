### What's new

- **folk-api v0.2.7** — docs: `PluginFactory::create` now carries an explicit doc comment stating that implementations must accept an empty JSON object `{}` and apply defaults ([#37](https://github.com/Folk-Project/folk-releases/issues/37))
  - `folk-builder` passes `{}` for every plugin section absent from `folk.toml`; this contract was previously implicit and undocumented, making it easy for a new plugin to accidentally require a field without a default
  - The doc comment also includes a copy-paste regression test snippet so new plugins can add the invariant in one line

- **folk-plugin-http v0.3.9**, **folk-plugin-jobs v0.3.4**, **folk-plugin-grpc v0.2.6**, **folk-plugin-metrics v0.2.3** — test: add `factory_accepts_empty_config` regression tests ([#37](https://github.com/Folk-Project/folk-releases/issues/37))
  - Each plugin now has a `tests/` regression test that calls `folk_plugin_factory().create(json!({}))` and asserts `Ok`, guarding against the class of bug that caused folk-plugin-process to crash on an absent `[process]` section (#36)

- **folk/spiral v0.1.2**, **folk/yii3 v0.1.1** — feat: surface `request_id` in application logs ([#38](https://github.com/Folk-Project/folk-releases/issues/38))
  - Completes the request_id logging integration started in Phase 45 (Laravel) and Phase 45 (Symfony); now all four PHP adapters stamp the UUID v7 request_id on app-side log entries
  - Monolog path (Spiral, and Yii3 when monolog is installed): `FolkRequestIdProcessor` reads `Folk::requestId()` at write time and adds `extra.request_id`; stateless, no per-request reset
  - Yii3 native path: `FolkBootstrap` reflection-wraps `Yiisoft\Log\Logger`'s private `contextProvider` with an anonymous class that merges `request_id` into every record's context; no hard dep on `yiisoft/log`
  - All paths are optional and silent-fail on `Throwable` — logger unavailability never crashes the worker
  - Smoke-tested in Docker (UUID v7 returned by `Folk::requestId()` in PHP matches Rust HTTP access log)


### Versions

| Package | Version | Type |
|---------|---------|------|
| folk-api | 0.2.7 | crates.io |
| folk-core | 0.3.6 | crates.io |
| folk-ext | 0.3.6 | crates.io |
| folk-plugin-http | 0.3.9 | crates.io |
| folk-plugin-grpc | 0.2.6 | crates.io |
| folk-plugin-jobs | 0.3.4 | crates.io |
| folk-plugin-metrics | 0.2.3 | crates.io |
| folk-plugin-process | 0.2.4 | crates.io |
| folk-builder | 0.2.4 | crates.io |
| folk-sdk | 0.3.0 | packagist |
| folk/laravel | 0.3.5 | packagist |
| folk/spiral | 0.1.2 | packagist |
| folk/yii3 | 0.1.1 | packagist |
