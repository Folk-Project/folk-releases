### 0.2.3 — lazy plugin loading

**Load a plugin only when its config section is present ([#80](https://github.com/Folk-Project/folk-releases/issues/80)) — `folk-builder` 0.2.12.**
A binary compiled with every plugin used to start them all: even a `folk.toml`
with only `[http]` still instantiated jobs/grpc/metrics/process with defaults,
which started and cluttered the logs.

Folk now follows a **"section present = plugin enabled"** model. A plugin is
instantiated only when its `folk.toml` section exists:

- **No section** → the plugin is not loaded (does not start, writes nothing to the log).
- **Empty section** (e.g. `[metrics]` with no options) → loaded with defaults.
- There is no `enabled = false` flag — omit the section to disable a plugin.

This lets you compile one binary with every plugin and switch each on or off
purely through config. A config with only `[http]` runs an HTTP-only server;
omitting `[http]` too is allowed (a jobs-only or gRPC-only server) and starts
silently without an HTTP listener.

```toml
[http]           # HTTP enabled
listen = "0.0.0.0:8080"
# no [jobs]/[grpc]/[metrics]/[process] → those plugins are not loaded
```

**Behavior change (semi-breaking).** Configs that relied on implicitly-enabled
plugins must now declare the section explicitly (e.g. add an empty `[metrics]`
to keep `/metrics` serving).

**`phpversion("folk")` reports the real version — `folk-builder` 0.2.13.**
It used to always return `0.1.0`: the builder-generated extension crate had a
hardcoded version, and PHP derives the extension version from it. The generated
crate is now stamped with the release version, so `phpversion("folk")` returns
`0.2.3` and tracks each release. (`\Folk\Sdk\Folk::version()` still reports the
`folk-ext` core version separately.)

**Prebuilt extension.** The `0.2.3` prebuilt `folk.so` is rebuilt with
`folk-builder` 0.2.13; no plugin crate changed.
