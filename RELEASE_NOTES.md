### What's new

- **folk-plugin-http v0.3.3** — fix: poisoned `X-Forwarded-For` header no longer allows client IP spoofing ([#60](https://github.com/Folk-Project/folk-releases/issues/60))
  - An unparseable entry in the XFF chain (e.g. `garbage, 10.0.0.1, 1.2.3.4`) was silently skipped, allowing a malicious client to bypass IP-based rate limiting or access control by injecting a fake trusted hop
  - Fix: an unparseable XFF entry is now treated as an untrusted boundary — the walk stops and `peer_ip` is returned instead of continuing left through attacker-controlled values

### What's new (previous)

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
| folk-api | 0.2.3 | crates.io |
| folk-core | 0.3.1 | crates.io |
| folk-ext | 0.3.1 | crates.io |
| folk-plugin-http | 0.3.3 | crates.io |
