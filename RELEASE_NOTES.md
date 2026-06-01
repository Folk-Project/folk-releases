### What's new

- **folk-api v0.2.1** — Fix broken tests, add execute_value coverage
  - Removed dead `execute_direct` tests (method was replaced by `execute_value` in phase 24)
  - Added `execute_value_default_roundtrips_via_json` test
  - `cargo test` now passes cleanly
  - Fixes [#14](https://github.com/Folk-Project/folk-releases/issues/14)

- **folk-core/folk-ext v0.2.8** — Stability fixes and cleanup
  - Replace `sleep(100ms)` with `Barrier` for deterministic tokio runtime synchronization — no more race condition on startup ([#8](https://github.com/Folk-Project/folk-releases/issues/8))
  - Make metric registration idempotent — duplicate names no longer crash the process ([#16](https://github.com/Folk-Project/folk-releases/issues/16))
  - Remove dead dependencies: `rmp-serde`, `rmpv`, `clap`, `tokio-util` ([#11](https://github.com/Folk-Project/folk-releases/issues/11))
  - Fix VCWD restore: move `chdir` to after `php_execute_script` so Composer proxy scripts (`vendor/bin/folk-server`) work correctly ([#27](https://github.com/Folk-Project/folk-releases/issues/27))
  - Remove unimplemented `max_memory_mb` config field — not feasible for ZTS thread workers ([#15](https://github.com/Folk-Project/folk-releases/issues/15))
  - Move inline `#[cfg(test)]` to `tests/` directory per project rules ([#29](https://github.com/Folk-Project/folk-releases/issues/29))

- **folk-plugin-http v0.2.3** — Config validation, read_timeout, compression.min_size, connection guard, h2c shutdown
  - Config deserialization errors now propagate instead of silently using defaults ([#7](https://github.com/Folk-Project/folk-releases/issues/7))
  - Implement `read_timeout`: request body reading now respects the configured timeout, returns HTTP 408 on expiry ([#18](https://github.com/Folk-Project/folk-releases/issues/18))
  - Implement `compression.min_size`: responses smaller than `min_size` are no longer compressed ([#18](https://github.com/Folk-Project/folk-releases/issues/18))
  - RAII `ConnectionGuard` for `active_connections` counter — no more permanent inflation on handler panic ([#19](https://github.com/Folk-Project/folk-releases/issues/19))
  - h2c graceful shutdown now waits for active connections via `JoinSet` instead of dropping them ([#20](https://github.com/Folk-Project/folk-releases/issues/20))

- **folk-plugin-grpc v0.2.4** — Health endpoint fix, base64 error handling
  - gRPC health endpoint now reports `SERVING` instead of `NOT_SERVING` ([#17](https://github.com/Folk-Project/folk-releases/issues/17))
  - Invalid base64 in PHP worker response now returns gRPC `INTERNAL` (13) with a warning log instead of silently passing garbage ([#23](https://github.com/Folk-Project/folk-releases/issues/23))

- **folk-plugin-jobs v0.3.2** — Redis password redacted in Debug
  - Custom `Debug` impl for `JobsConfig` masks the password field as `[REDACTED]` ([#21](https://github.com/Folk-Project/folk-releases/issues/21))

- **folk-plugin-metrics v0.2.2** — Config validation
  - Config deserialization errors now propagate instead of silently using defaults ([#7](https://github.com/Folk-Project/folk-releases/issues/7))

- **folk-plugin-process v0.2.2** — MessagePack removal, error handling, kill() fix
  - Replace MessagePack with JSON in `process.list` and `process.restart` RPCs — PHP clients can now decode responses ([#12](https://github.com/Folk-Project/folk-releases/issues/12))
  - `process.restart` returns `Err` for unknown process names instead of `Ok` with error text ([#30](https://github.com/Folk-Project/folk-releases/issues/30))
  - Config deserialization errors now propagate instead of silently using defaults ([#7](https://github.com/Folk-Project/folk-releases/issues/7))
  - `libc::kill()` return value is now checked; added `#[cfg(not(unix))]` fallback via `child.kill()` ([#22](https://github.com/Folk-Project/folk-releases/issues/22))
  - Removed `rmp-serde` dependency

- **folk-builder v0.2.1** — Config validation in generated code
  - Generated `create_plugins()` now propagates `serde_json::to_value` errors instead of silently falling back to defaults ([#7](https://github.com/Folk-Project/folk-releases/issues/7))

### Versions

| Package | Version | Type |
|---------|---------|------|
| folk-api | 0.2.1 | crates.io |
| folk-core | 0.2.8 | crates.io |
| folk-ext | 0.2.8 | crates.io |
| folk-plugin-http | 0.2.3 | crates.io |
| folk-plugin-grpc | 0.2.4 | crates.io |
| folk-plugin-jobs | 0.3.2 | crates.io |
| folk-plugin-metrics | 0.2.2 | crates.io |
| folk-plugin-process | 0.2.2 | crates.io |
| folk-builder | 0.2.1 | crates.io |
