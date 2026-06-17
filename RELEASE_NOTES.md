### What's new

- **folk-core / folk-ext v0.2.10, folk-sdk v0.2.7, folk-builder v0.2.3** — Request IDs ([#34](https://github.com/Folk-Project/folk-releases/issues/34))
  - Every request now carries a unique, monotonic `request_id` threaded end-to-end from the Rust dispatcher into the PHP worker
  - New native function `folk_request_id()` and SDK facade `\Folk\Sdk\Folk::requestId()` return the current request's id (0 outside a request) — use it to correlate PHP application logs with Folk's Rust-side access logs
  - New `[workers]` setting `max_concurrent_per_worker` (default `1`). Only `1` is supported today; values `> 1` are reserved for a future async runtime (PHP Fibers) and are clamped to `1` with a warning. No behavior change for existing setups
  - Foundation for upcoming response streaming ([#35](https://github.com/Folk-Project/folk-releases/issues/35))

- **folk-plugin-process v0.2.3** — Empty/absent `[process]` config no longer crashes ([#36](https://github.com/Folk-Project/folk-releases/issues/36))
  - A config with no `[process]` section (or an empty one) previously failed on startup with `Config error: invalid [process] configuration`. The process plugin is always compiled in, so an absent section is a normal "supervise nothing" case — it now starts cleanly, like the other plugins
  - Config errors now surface the underlying reason (e.g. `missing field \`command\``) instead of a generic message

### Versions

| Package | Version | Type |
|---------|---------|------|
| folk-core | 0.2.10 | crates.io |
| folk-ext | 0.2.10 | crates.io |
| folk-sdk | 0.2.7 | packagist |
| folk-builder | 0.2.3 | crates.io |
| folk-plugin-process | 0.2.3 | crates.io |
