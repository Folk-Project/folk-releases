### What's new

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

### Versions

| Package | Version | Type |
|---------|---------|------|
| folk-core | 0.2.7 | crates.io |
| folk-ext | 0.2.7 | crates.io |
| folk/sdk | 0.2.4 | packagist |
| folk/laravel | 0.2.3 | packagist |
| folk-plugin-http | 0.2.2 | crates.io |
| folk-plugin-jobs | 0.3.0 | crates.io |
| folk-plugin-grpc | 0.2.3 | crates.io |
| folk-plugin-metrics | 0.2.1 | crates.io |
| folk-plugin-process | 0.2.1 | crates.io |
