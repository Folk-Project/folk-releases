# Release Notes

### Plugins own their PHP host-function surface by convention ([#91](https://github.com/Folk-Project/folk-releases/issues/91))

**A maintenance/architecture release — no change to the PHP API or behaviour.** The
extension exposes the same `folk_*` functions as before.

Follow-up to [#89](https://github.com/Folk-Project/folk-releases/issues/89): that
release moved gRPC's `folk_grpc_descriptors` body into the plugin, but folk-builder
still referenced the gRPC plugin **by name** to wire it in. Now the builder is fully
**name-agnostic**:

- **folk-builder 0.2.20** — the generated extension calls
  `<plugin>::register_php_functions(module)` for **every** plugin whose build enables
  the `php-ext` feature, in config order. No plugin is special-cased by name; the
  `folk_plugin_grpc` string is gone from the codegen.

**Convention for plugin authors** (new [Plugin Development](https://folk-project.github.io/folk-releases/plugin-development/)
doc): a plugin that needs its own PHP function exposes an optional `php-ext` Cargo
feature and a `pub fn register_php_functions(ModuleBuilder) -> ModuleBuilder`, then
is declared with `features = ["php-ext"]`. Adding a PHP host function to a new plugin
no longer requires touching folk-builder.

!!! note "Building your own `.so` with gRPC"
    Because the builder no longer auto-injects the feature, a `folk.build.toml` (or
    the release `build-manifest.toml`) that includes `folk-plugin-grpc` must now
    declare `features = ["php-ext"]` to keep `folk_grpc_descriptors` /
    `php artisan folk:grpc:generate`. See
    [Installation → Build from Source](https://folk-project.github.io/folk-releases/installation/#build-from-source).

The [Installation](https://folk-project.github.io/folk-releases/installation/#build-from-source)
guide now documents the full `folk.build.toml` schema (all `[[plugin]]` fields) so a
custom build is self-service.

Prebuilt extension release `0.2.12` (folk-builder 0.2.20; gRPC declared with
`features = ["php-ext"]`). No plugin crate versions changed.
