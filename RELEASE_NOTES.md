### What's new

- **folk-core/folk-ext v0.2.7** — Critical bugfix: `execute_script` return value normalization
  - `folk_zts_execute_script` now explicitly returns 1 (success) / 0 (failure), matching `folk_zts_eval_string` convention
  - Previously the raw `php_execute_script` result was passed through without normalization — PHP workers could not detect real script startup errors
  - Fixed misleading `fopen` failure path that returned 0 with a "SUCCESS" comment
  - Rust FFI wrapper unchanged — `if ret != 0 { Ok(()) }` was already correct for the normalized convention
  - Fixes [#1](https://github.com/Folk-Project/folk-releases/issues/1)

### Versions

| Package | Version | Type |
|---------|---------|------|
| folk-core | 0.2.7 | crates.io |
| folk-ext | 0.2.7 | crates.io |
| folk/sdk | 0.2.4 | packagist |
| folk/laravel | 0.2.3 | packagist |
| folk-plugin-http | 0.2.2 | crates.io |
| folk-plugin-jobs | 0.3.0 | crates.io |
| folk-plugin-grpc | 0.2.2 | crates.io |
| folk-plugin-metrics | 0.2.1 | crates.io |
| folk-plugin-process | 0.2.1 | crates.io |
