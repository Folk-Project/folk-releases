### What's new

- **folk-plugin-http v0.3.1** — bugfix: binary response bodies corrupted with Lua hooks ([#49](https://github.com/Folk-Project/folk-releases/issues/49))
  - Binary HTTP responses (images, PDFs, gzip, protobuf, `application/octet-stream`) were silently corrupted on every request when any `[[http.hooks]]` entry was present in `folk.toml`, even if the hook only handled `request.before` and never touched the response
  - Root cause: response body was unconditionally converted via `String::from_utf8_lossy` (replacing invalid UTF-8 bytes with U+FFFD) for all hooks, not only `response.after`
  - Fix: body is now materialised only when `response.after` hooks are actually registered; all other hook events leave the original bytes intact


### Versions

| Package | Version | Type |
|---------|---------|------|
| folk-plugin-http | 0.3.1 | crates.io |
