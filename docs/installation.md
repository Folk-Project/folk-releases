# Installation

## Requirements

- PHP 8.2+ — the standard **NTS** (non-thread-safe) build. Folk forks worker
  processes, so multi-worker needs no thread safety.
- Linux x86_64/aarch64 or macOS Apple Silicon

## Via PIE (recommended)

[PIE](https://github.com/php/pie) is the modern PHP extension installer from the PHP Foundation.

```bash
pie install folk-project/ext-folk
```

PIE will automatically download a pre-built binary for your platform. No Rust toolchain needed.

## Via Docker

```dockerfile
FROM php:8.4

RUN apt-get update && apt-get install -y unzip curl \
    && rm -rf /var/lib/apt/lists/*

RUN docker-php-ext-install pcntl sockets

RUN curl -Lo /usr/local/bin/pie https://github.com/php/pie/releases/latest/download/pie.phar \
    && chmod +x /usr/local/bin/pie \
    && pie install folk-project/ext-folk

COPY --from=composer:2 /usr/bin/composer /usr/bin/composer

WORKDIR /app
COPY . .
RUN composer install --no-dev

# Framework adapters (Laravel / Spiral / Symfony / Yii 3) all expose the same
# Composer bin-proxy — one command regardless of framework:
CMD ["php", "vendor/bin/folk-server"]
# For plain SDK:
# CMD ["php", "vendor/bin/folk-worker"]
```

## Manual install

Download the matching ZIP from [GitHub Releases](https://github.com/Folk-Project/folk-releases/releases), extract, and copy to your PHP extension directory:

```bash
cp folk.so $(php -r "echo ini_get('extension_dir');")/
echo "extension=folk" > $(php --ini | grep 'Scan' | awk '{print $NF}')/folk.ini
```

## Build from Source

If you need a custom plugin set or want to build locally:

### 1. Install Rust

```bash
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
```

### 2. Install build dependencies

=== "Ubuntu/Debian"

    ```bash
    apt-get install pkg-config libclang-dev clang protobuf-compiler
    ```

=== "macOS"

    ```bash
    brew install llvm protobuf
    ```

### 3. Install folk-builder

```bash
cargo install folk-builder
```

### 4. Create `folk.build.toml`

`folk.build.toml` lists which plugins are compiled into your `.so`. One
`[[plugin]]` block per plugin:

```toml
[build]
output = "folk"                     # produces folk.so

[[plugin]]
crate_name = "folk-plugin-http"     # crates.io crate name
version = "0.5"                     # semver req (Cargo syntax, e.g. "0.5", "0.5.*")
config_key = "http"                 # the folk.toml section that configures it ([http])

[[plugin]]
crate_name = "folk-plugin-jobs"
version = "0.7"
config_key = "jobs"
features = ["redis", "embedded"]    # optional: Cargo features on the plugin

[[plugin]]
crate_name = "folk-plugin-grpc"
version = "0.7"
config_key = "grpc"
features = ["php-ext"]              # REQUIRED for gRPC — see note below
```

Per-plugin fields:

| Field | Meaning |
|-------|---------|
| `crate_name` | The plugin crate (from crates.io). |
| `version` | Cargo version requirement. Omit if using `path`/`git`. |
| `path` / `git` | Build from a local path or git repo instead of crates.io (for development). |
| `config_key` | The `folk.toml` section name that enables/configures the plugin. |
| `features` | Cargo features to enable on the plugin crate. |

!!! important "gRPC needs the `php-ext` feature"
    The gRPC plugin exposes the PHP host function `folk_grpc_descriptors` (used by
    proto → DTO code generation, e.g. `php artisan folk:grpc:generate`) behind its
    **`php-ext`** Cargo feature. If you include `folk-plugin-grpc`, add
    `features = ["php-ext"]` — otherwise that function is left out of the `.so` and
    generation fails. This is the general convention: a plugin that adds its own PHP
    functions declares `features = ["php-ext"]`, and the builder wires it in
    automatically (see [Plugin Development](plugin-development.md)).

### 5. Build

```bash
folk-builder build --config folk.build.toml --output-dir .
```

This produces `folk.so`. Copy it to your PHP extension directory:

```bash
cp folk.so $(php -r "echo ini_get('extension_dir');")
echo "extension=folk" > $(php --ini | grep 'Scan' | awk '{print $NF}')/folk.ini
```

## Verify Installation

```bash
php -m | grep folk
# Should output: folk

php -r "echo folk_version();"
```
