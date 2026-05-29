### What's new

- **folk-core/folk-ext v0.2.6** — Automatic opcache warmup (phase 41):
  - Before spawning worker threads, Folk automatically compiles all files from Composer classmap into shared opcache
  - Uses `opcache_compile_file()` — safe compilation without code execution, no side effects
  - New `zts::eval_string()` FFI wrapper (`zend_eval_string`) — no temp files needed
  - ~10x faster worker cold start (compile step eliminated, only symbol registration remains)
  - Works with any framework (Laravel, Symfony, Yii, vanilla PHP) — no configuration needed
  - `warmup = false` in `[workers]` to disable if needed (default: enabled)
  - Graceful degradation: if no Composer classmap found or opcache disabled, warmup is skipped with a warning

### Versions

| Package | Version | Type |
|---------|---------|------|
| folk-core | 0.2.6 | crates.io |
| folk-ext | 0.2.6 | crates.io |
| folk/sdk | 0.2.4 | packagist |
| folk/laravel | 0.2.3 | packagist |
| folk-plugin-http | 0.2.2 | crates.io |
| folk-plugin-jobs | 0.3.0 | crates.io |
| folk-plugin-grpc | 0.2.2 | crates.io |
| folk-plugin-metrics | 0.2.1 | crates.io |
| folk-plugin-process | 0.2.1 | crates.io |
