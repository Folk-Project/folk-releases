# Installation

## Requirements

- PHP 8.2+ (**NTS** — non-thread-safe; ZTS is no longer required). Folk forks
  worker processes, so multi-worker needs no thread safety.
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

# For Laravel:
CMD ["php", "vendor/folk/laravel/bin/folk-server"]
# For Spiral:
# CMD ["php", "vendor/folk/spiral/bin/folk-server"]
# For Symfony:
# CMD ["php", "vendor/folk/symfony/bin/folk-server"]
# For Yii 3:
# CMD ["php", "vendor/folk/yii3/bin/folk-server"]
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
echo "extension=folk" > $(php --ini | grep 'Scan' | awk '{print $NF}')/folk.ini
```

## Verify Installation

```bash
php -m | grep folk
# Should output: folk

php -r "echo folk_version();"
```
