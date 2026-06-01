### What's new

- **folk-plugin-http v0.2.4** — Fix binary body corruption ([#13](https://github.com/Folk-Project/folk-releases/issues/13))
  - Binary HTTP bodies (multipart/form-data, application/octet-stream) were irreversibly corrupted by `from_utf8_lossy`
  - Now uses base64 fallback: if body is not valid UTF-8, encodes as base64 and adds `body_encoding` field
  - Response path also supports `body_encoding: "base64"` from PHP side

- **folk-sdk v0.2.6** — Base64 body encoding support ([#13](https://github.com/Folk-Project/folk-releases/issues/13))
  - `HttpRequest::fromPayload()` decodes base64 body when `body_encoding === "base64"`
  - `HttpResponse::toPayload()` encodes non-UTF-8 bodies as base64 automatically

- **folk-builder v0.2.2** — Path canonicalization + version fix ([#25](https://github.com/Folk-Project/folk-releases/issues/25), [#26](https://github.com/Folk-Project/folk-releases/issues/26))
  - `folk_api_path` is now canonicalized before TempDir build — relative paths no longer break `cargo build`
  - CLI version uses `env!("CARGO_PKG_VERSION")` instead of hardcoded `"0.1.0"`

### Versions

| Package | Version | Type |
|---------|---------|------|
| folk-plugin-http | 0.2.4 | crates.io |
| folk-builder | 0.2.2 | crates.io |
| folk-sdk | 0.2.6 | packagist |
