# Installation

## Requirements

- PHP 8.3+ with **ZTS** (Zend Thread Safety) for multi-worker mode
- Linux x86_64 or aarch64 (macOS support planned)

## Via Composer (recommended)

```bash
composer require folk/sdk
```

The SDK will automatically download the pre-built `folk.so` extension matching your platform.

For Laravel projects:

```bash
composer require folk/laravel
```

## Via Docker

The simplest way to get started. No local Rust toolchain needed.

```dockerfile
FROM php:8.4-zts AS builder

RUN curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs \
    | sh -s -- -y --default-toolchain 1.88.0
ENV PATH="/root/.cargo/bin:${PATH}"

RUN apt-get update && apt-get install -y \
    pkg-config libclang-dev clang protobuf-compiler \
    && rm -rf /var/lib/apt/lists/*

RUN cargo install folk-builder@0.2.0

WORKDIR /build
COPY folk.build.toml .
RUN folk-builder build --config folk.build.toml --output-dir /build/

FROM php:8.4-zts
COPY --from=builder /build/folk.so /usr/local/lib/php/extensions/folk.so
RUN echo "extension=/usr/local/lib/php/extensions/folk.so" \
    > /usr/local/etc/php/conf.d/folk.ini

COPY . /app
WORKDIR /app
CMD ["php", "vendor/bin/folk-worker"]
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

```toml
[build]
output = "folk"

[[plugin]]
crate_name = "folk-plugin-http"
version = "0.2"
config_key = "http"

[[plugin]]
crate_name = "folk-plugin-jobs"
version = "0.2"
config_key = "jobs"
```

### 5. Build

```bash
folk-builder build --config folk.build.toml --output-dir .
```

This produces `folk.so`. Copy it to your PHP extension directory:

```bash
cp folk.so $(php -r "echo ini_get('extension_dir');")
echo "extension=folk.so" > $(php --ini | grep 'Scan' | awk '{print $NF}')/folk.ini
```

## Verify Installation

```bash
php -m | grep folk
# Should output: folk

php -r "echo folk_version();"
# Should output: folk-ext 0.2.3
```
