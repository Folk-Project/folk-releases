### What's new

- **folk-plugin-http v0.3.0** — Lua hook pipeline ([#33](https://github.com/Folk-Project/folk-releases/issues/33))
  - Attach Lua scripts to HTTP request lifecycle events via `[[http.hooks]]` in `folk.toml` — no Rust code required
  - Four hook events: `request.before` (before PHP dispatch), `request.error` (PHP error/timeout), `response.headers` (after PHP headers), `response.after` (full PHP response)
  - `sync` hooks run in the critical path and can **short-circuit** — return an HTTP response without ever reaching PHP. Rate limiting, auth checks, and routing rules run at Rust speed
  - `async` hooks fire-and-forget outside the critical path — ideal for audit logging and metrics
  - Per-hook `on_error`: `fail_open` (default — skip on error, log WARN) or `fail_closed` (return 500 on error). Use `fail_closed` for security hooks where a failing script must not silently pass requests through
  - Scripts that fail to compile at startup are skipped with WARN; the server starts normally
  - Powered by [mlua](https://github.com/mlua-rs/mlua) with embedded Lua 5.4 (no system Lua required)


### Versions

| Package | Version | Type |
|---------|---------|------|
| folk-plugin-http | 0.3.0 | crates.io |
