### What's new

- **folk-core / folk-ext v0.3.2** — fix: env var overrides with multi-word field names now work correctly ([#58](https://github.com/Folk-Project/folk-releases/issues/58))
  - `FOLK_WORKERS_MAX_JOBS`, `FOLK_HTTP_WRITE_TIMEOUT` and similar multi-word env vars were silently ignored — `split("_")` mapped `FOLK_SERVER_SHUTDOWN_TIMEOUT` to `server.shutdown.timeout` (no such path) instead of `server.shutdown_timeout`
  - Fix: the section separator is now **double underscore** (`__`): `FOLK_WORKERS__MAX_JOBS`, `FOLK_HTTP__WRITE_TIMEOUT`, `FOLK_SERVER__SHUTDOWN_TIMEOUT`
  - **Breaking**: single-underscore env vars that previously happened to work for single-word fields (e.g. `FOLK_WORKERS_COUNT`) must be updated to `FOLK_WORKERS__COUNT`

- **folk-plugin-http v0.3.4** — fix: `request.error` hooks now respect the `mode` field ([#50](https://github.com/Folk-Project/folk-releases/issues/50))
  - `run_request_error` was unconditionally spawning every matching hook via `tokio::spawn`, silently ignoring the configured `mode` and `on_error` fields
  - A hook with `mode = "sync"` was run as fire-and-forget async; `on_error = "fail_closed"` had no effect
  - Fix: `request.error` hooks are now partitioned the same way as `request.before` — sync hooks run inline (critical path) with full short-circuit and `fail_closed` support, async hooks fire-and-forget

- **folk-plugin-http v0.3.3** — fix: poisoned `X-Forwarded-For` header no longer allows client IP spoofing ([#60](https://github.com/Folk-Project/folk-releases/issues/60))
  - An unparseable entry in the XFF chain (e.g. `garbage, 10.0.0.1, 1.2.3.4`) was silently skipped, allowing a malicious client to bypass IP-based rate limiting or access control by injecting a fake trusted hop
  - Fix: an unparseable XFF entry is now treated as an untrusted boundary — the walk stops and `peer_ip` is returned instead of continuing left through attacker-controlled values

- **folk-api v0.2.4** — fix: `ServerPluginWrapper::boot()` now returns an error if called twice while the plugin is already running ([#54](https://github.com/Folk-Project/folk-releases/issues/54))
  - Previously the old `JoinHandle` was silently overwritten without `.abort()`, leaving the original task running as a ghost and sharing resources with the replacement
  - Fix: `boot()` checks if a handle is already present and returns `Err` immediately — double-boot is a programming error, not a recoverable condition

- **folk-api v0.2.3** — fix: `ServerPluginWrapper::shutdown()` no longer deadlocks when a plugin task ignores `ctx.shutdown` ([#55](https://github.com/Folk-Project/folk-releases/issues/55))
  - Previously, a plugin task stuck in I/O or waiting on a channel blocked `handle.await` forever, making SIGTERM unresponsive
  - Fix: `handle.abort()` is called before `handle.await`; `JoinError::Cancelled` is treated as a clean exit
  - The outer `[server] shutdown_timeout` (default 30 s) remains the overall deadline

- **folk-core / folk-ext v0.3.1** — fix: dead worker slots no longer block requests ([#57](https://github.com/Folk-Project/folk-releases/issues/57))
  - When a slot's supervisor task crashed (panic or unexpected exit), its inbox receiver was dropped; the round-robin dispatcher kept sending to that slot, failing 1/N of all requests with a misleading error
  - Fix: on `SendError` the slot is marked dead, the request value is recovered from the error, and the next live slot is tried immediately
  - If all slots are dead, the caller now receives an explicit `"all worker slots dead"` error instead of a silently dropped reply channel

- **folk-plugin-http v0.3.2** — complete fix for binary response body corruption with Lua hooks ([#49](https://github.com/Folk-Project/folk-releases/issues/49))
  - Binary HTTP responses (images, PDFs, gzip, protobuf, `application/octet-stream`) were silently corrupted whenever any `[[http.hooks]]` entry was present in `folk.toml`
  - Root cause: `ResponseContext.body` was `Option<String>` and populated via `String::from_utf8_lossy`, replacing invalid UTF-8 bytes with U+FFFD — even inside `response.after` hooks
  - Fix: `ResponseContext.body` is now `Option<Vec<u8>>`; bytes are passed to Lua as a Lua byte string (`lua.create_string`) and read back via `mlua::String::as_bytes()` — no UTF-8 conversion at any point
  - Binary bodies now pass through all hook events byte-for-byte regardless of content type


### Versions

| Package | Version | Type |
|---------|---------|------|
| folk-api | 0.2.4 | crates.io |
| folk-core | 0.3.2 | crates.io |
| folk-ext | 0.3.2 | crates.io |
| folk-plugin-http | 0.3.4 | crates.io |
