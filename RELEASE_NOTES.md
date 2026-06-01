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

### Versions

| Package | Version | Type |
|---------|---------|------|
| folk-api | 0.2.1 | crates.io |
| folk-core | 0.2.8 | crates.io |
| folk-ext | 0.2.8 | crates.io |
