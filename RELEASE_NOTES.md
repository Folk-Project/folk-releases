# Release Notes

### Jobs plugin: driver registry cleanup ([#77](https://github.com/Folk-Project/folk-releases/issues/77))

Internal maintainability release for `folk-plugin-jobs` — **no behavior, config, or
API change**. The prebuilt extension is rebuilt against the refactored crate; if you use
Folk through the published `.so` or via crates.io, nothing about your `folk.toml`,
queues, drivers, or dispatch changes.

**What changed under the hood.** Adding or maintaining a queue backend used to touch
five scattered places, most of it near-duplicated `#[cfg(feature = …)]` boilerplate
(a `compiled_drivers()` list, a `build_<driver>()` function per backend, an
`unavailable()` helper, and a routing arm each). All of that is now generated from a
single declarative driver table (`drivers!` in `src/registry.rs`). Adding a backend is
now: a driver module, one config field, and **one table row** — ~120 lines of `cfg`
duplication removed.

**Why not split into per-driver crates** (the original #77 proposal). Lean builds are
already delivered by Cargo features (`features = ["redis"]` never compiles the
Kafka/AWS/GCP SDKs), and the prebuilt `.so` ships every driver regardless. A crate split
would only buy independent crates.io versioning at the cost of 8× publish surface — not
worth it for the one real pain point, which is fixed in place here.

**Protected behavior — unchanged and verified:** the `[jobs.connections.<driver>]` TOML
layout, secret redaction in logs, queue routing (`[<connection>.]<queue>`), the
feature-missing fail-fast error, the `jobs.push` / `jobs.stats` RPC, metric names, and
the consume / retry / dead-letter loop. The full unit + integration suite passes without
test changes; end-to-end job push→redis→consume was smoke-tested on Laravel (NTS fork
model).

**Versions:** folk-plugin-jobs **0.7.1** (crates.io). No other crate changed — folk-api
(0.3.3), folk-core/folk-ext (0.6.3), folk-plugin-grpc (0.7.1), folk-builder (0.2.20), and
all PHP adapters are untouched. Prebuilt extension release `0.2.14`.
