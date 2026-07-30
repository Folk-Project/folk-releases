# Plugin Development

A Folk plugin is a Rust crate that implements the [`folk_api::Plugin`] trait and is
compiled into the extension by [folk-builder](installation.md#build-from-source).
The built-in plugins (HTTP, gRPC, Jobs, Metrics, Process) are ordinary crates — a
third-party plugin looks exactly the same.

This page focuses on the one thing that isn't obvious: **how a plugin talks to
PHP**.

## Two ways a plugin reaches PHP

### 1. The RPC channel (default — no PHP function needed)

Most plugins never define their own PHP function. They register RPC handlers in
`boot()` via the `RpcRegistrar`, and PHP calls them through the single generic host
function `folk_call("<plugin>.<method>", $payload)` (provided by folk-ext, always
available). This is how `jobs.push`, `metrics.*`, `grpc.client.call`, etc. work —
one bridge, any number of methods, zero per-plugin PHP surface.

**Prefer this.** If your plugin's PHP-facing work happens during a request or can be
modelled as "PHP calls a method, gets a result", use an RPC handler.

### 2. A dedicated PHP host function (the rare case)

Occasionally a plugin needs a real PHP function — a global like
`my_plugin_do_thing(...)` — because it runs **outside** the request/worker dispatch
model. The only built-in example is gRPC's `folk_grpc_descriptors($paths)`, a
build/codegen-time utility that compiles `.proto` files to a descriptor set
synchronously, with no server running.

If you need this, follow the **`php-ext` convention**. The builder is
name-agnostic: it calls `register_php_functions` for **any** plugin whose build
enables the `php-ext` feature — you do **not** need to modify folk-builder.

#### Step 1 — an optional `php-ext` feature

```toml
# Cargo.toml
[features]
# Compile the PHP host-function surface. Off by default so the plugin stays
# PHP-runtime-agnostic; folk-builder enables it when the plugin is declared with
# features = ["php-ext"].
php-ext = ["dep:ext-php-rs"]

[dependencies]
ext-php-rs = { version = "0.15", optional = true }
```

#### Step 2 — the functions + a `register_php_functions` registrar

```rust
// src/php_ext.rs  (compiled only under feature = "php-ext")
use ext_php_rs::binary::Binary;
use ext_php_rs::prelude::*;

#[php_function]
#[allow(clippy::needless_pass_by_value)]
pub fn my_plugin_do_thing(paths: Vec<String>) -> PhpResult<Binary<u8>> {
    let out = crate::do_thing(&paths)
        .map_err(|e| PhpException::default(format!("my_plugin_do_thing: {e}")))?;
    Ok(Binary::new(out))
}

/// Register this plugin's PHP host functions. folk-builder calls this from the
/// generated `get_module` for every plugin built with the `php-ext` feature.
pub fn register_php_functions(module: ModuleBuilder) -> ModuleBuilder {
    module.function(wrap_function!(my_plugin_do_thing))
}
```

```rust
// src/lib.rs
#[cfg(feature = "php-ext")]
mod php_ext;
#[cfg(feature = "php-ext")]
pub use php_ext::register_php_functions;
```

!!! note "`wrap_function!` must expand inside your crate"
    `wrap_function!` only accepts a bare identifier and refers to a macro-generated
    companion symbol, so the registration must live in the same crate/module as the
    `#[php_function]`. That's exactly what `register_php_functions` is for — the
    builder just calls it.

#### Step 3 — declare the feature in the build

In `folk.build.toml` (or the release `build-manifest.toml`):

```toml
[[plugin]]
crate_name = "my-plugin"
version = "0.1"
config_key = "myplugin"
features = ["php-ext"]
```

That's it. folk-builder emits `my_plugin::register_php_functions(module)` into the
generated extension — no builder changes, no hardcoded plugin names. Without the
feature, your plugin is a normal RPC-only plugin and the function is absent.

Reference implementation: `folk-plugin-grpc/src/php_ext.rs`.

[`folk_api::Plugin`]: https://docs.rs/folk-api
