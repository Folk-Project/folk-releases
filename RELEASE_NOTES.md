### What's new

- **folk-api v0.2.1** — Fix broken tests, add execute_value coverage
  - Removed dead `execute_direct` tests (method was replaced by `execute_value` in phase 24)
  - Added `execute_value_default_roundtrips_via_json` test
  - `cargo test` now passes cleanly
  - Fixes [#14](https://github.com/Folk-Project/folk-releases/issues/14)

- **folk-core/folk-ext v0.2.7** — Critical bugfix: `execute_script` return value normalization
  - `folk_zts_execute_script` now explicitly returns 1 (success) / 0 (failure), matching `folk_zts_eval_string` convention
  - Previously the raw `php_execute_script` result was passed through without normalization — PHP workers could not detect real script startup errors
  - Fixed misleading `fopen` failure path that returned 0 with a "SUCCESS" comment
  - Fixes [#1](https://github.com/Folk-Project/folk-releases/issues/1)

- **folk-plugin-grpc v0.2.3** — Security fix: zip bomb prevention + status code correction
  - `gzip_decompress` now enforces `max_recv_message_size` **during** decompression via `.take(limit + 1)`, preventing OOM from malicious payloads
  - Fixed gRPC status code: `RESOURCE_EXHAUSTED` is **8**, not 11 (`OUT_OF_RANGE`)
  - Removed stray `folk-plugin-http/` directory from the repository
  - Fixes [#2](https://github.com/Folk-Project/folk-releases/issues/2), [#6](https://github.com/Folk-Project/folk-releases/issues/6), [#24](https://github.com/Folk-Project/folk-releases/issues/24)

- **folk-plugin-jobs v0.3.1** — Atomic job promotion + deadlock fix
  - RedisDriver: `promote_delayed` now uses a **Lua script** for atomic ZRANGEBYSCORE → RPUSH → ZREM (no more job loss/duplication on crash)
  - MemoryDriver: split lock scopes to prevent ABBA deadlock between `delayed` and `queues` mutexes
  - Errors in RedisDriver are now propagated instead of silently swallowed
  - Fixes [#3](https://github.com/Folk-Project/folk-releases/issues/3), [#4](https://github.com/Folk-Project/folk-releases/issues/4)

### Versions

| Package | Version | Type |
|---------|---------|------|
| folk-api | 0.2.1 | crates.io |
| folk-core | 0.2.7 | crates.io |
| folk-ext | 0.2.7 | crates.io |
| folk/sdk | 0.2.4 | packagist |
| folk/laravel | 0.2.3 | packagist |
| folk-plugin-http | 0.2.2 | crates.io |
| folk-plugin-jobs | 0.3.1 | crates.io |
| folk-plugin-grpc | 0.2.3 | crates.io |
| folk-plugin-metrics | 0.2.1 | crates.io |
| folk-plugin-process | 0.2.1 | crates.io |
