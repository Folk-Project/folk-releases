### What's new

- **HTTP plugin v0.2.1** — production-ready configuration:
  - `write_timeout` now enforced (returns 504 on timeout)
  - `max_request_size` — configurable body limit with human-readable format (`"10mb"`, `"512kb"`)
  - `access_log` — structured HTTP logging (client IP, method, URI, status, duration)
  - `trusted_proxies` — correct X-Forwarded-For extraction behind load balancers
  - TLS/SSL via rustls (`[http.tls]` with cert/key)
  - HTTP/2 cleartext (`h2c = true`)
  - Response compression: gzip, brotli, zstd, deflate (`[http.compression]`)

### Crate versions

| Crate | Version |
|-------|---------|
| folk-plugin-http | 0.2.1 |
| folk-plugin-jobs | 0.2.0 |
| folk-plugin-grpc | 0.2.0 |
| folk-plugin-metrics | 0.2.0 |
| folk-plugin-process | 0.2.0 |
| folk-core | 0.2.3 |
| folk-ext | 0.2.3 |
