### What's new

- **folk-api v0.2.1** — Fix broken tests, add execute_value coverage
  - Removed dead `execute_direct` tests (method was replaced by `execute_value` in phase 24)
  - Added `execute_value_default_roundtrips_via_json` test
  - `cargo test` now passes cleanly
  - Fixes [#14](https://github.com/Folk-Project/folk-releases/issues/14)

### Versions

| Package | Version | Type |
|---------|---------|------|
| folk-api | 0.2.1 | crates.io |
