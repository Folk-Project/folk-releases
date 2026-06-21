### What's new

- **folk-plugin-http v0.3.1** — bugfix: binary response bodies corrupted with Lua hooks ([#49](https://github.com/Folk-Project/folk-releases/issues/49))
  - Binary HTTP responses (images, PDFs, gzip, protobuf, `application/octet-stream`) were silently corrupted on every request when any `[[http.hooks]]` entry was present in `folk.toml`, even if the hook only handled `request.before` and never touched the response
  - Root cause: response body was unconditionally converted via `String::from_utf8_lossy` (replacing invalid UTF-8 bytes with U+FFFD) for all hooks, not only `response.after`
  - Fix: body is now materialised only when `response.after` hooks are actually registered; all other hook events leave the original bytes intact

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
| folk-plugin-http | 0.3.1 | crates.io |
