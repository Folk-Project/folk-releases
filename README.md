# Folk

**High-performance PHP application server powered by Rust.**

Folk replaces nginx + php-fpm with a single binary that handles HTTP, gRPC, background jobs, metrics, and managed processes — all with zero-copy communication between Rust and PHP.

## Documentation

📖 **[folk-project.github.io/folk-releases](https://folk-project.github.io/folk-releases/)**

## Quick Start

```bash
composer require folk/sdk
```

Create `folk.toml`:

```toml
[workers]
script = "vendor/bin/folk-worker"
count = 4

[http]
listen = "0.0.0.0:8080"
```

Run:

```bash
php vendor/bin/folk-worker
```

## Downloads

Pre-built extensions are available on the [Releases](https://github.com/Folk-Project/folk-releases/releases) page.

## Contributing

Found a bug or have an idea? See [Contributing Guide](https://folk-project.github.io/folk-releases/contributing/) for how to report issues and propose features.

All issues are tracked in this repository: [Issues](https://github.com/Folk-Project/folk-releases/issues)

## License

MIT
