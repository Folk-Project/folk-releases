### What's new

- **folk-core / folk-ext v0.2.9** — Hot reload / dev watch mode ([#28](https://github.com/Folk-Project/folk-releases/issues/28))
  - New `[dev]` config: `watch`, `watch_paths`, `watch_extensions`, `debounce`. Disabled by default
  - A file watcher recycles workers when watched `.php` files change — code edits are picked up without a manual restart; in-flight requests complete first (graceful)
  - Requires PHP ZTS with `workers.count > 1`: the main PHP thread (worker #1) is not recyclable, so single-worker servers cannot fully hot reload
  - No OPcache reset on reload (it raced live worker recycling and could crash). Keep `opcache.validate_timestamps = 1` in dev (default) so edits are seen
  - Also: forward-compat with `ext-php-rs` 0.15.15 (`ArrayKey::ZendString`)

### Versions

| Package | Version | Type |
|---------|---------|------|
| folk-core | 0.2.9 | crates.io |
| folk-ext | 0.2.9 | crates.io |
