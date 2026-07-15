# Folk

**High-performance PHP application server powered by Rust.**

Folk replaces nginx + php-fpm with a single binary that handles HTTP, gRPC, background jobs, metrics, and managed processes — all with zero-copy communication between Rust and PHP.

## Documentation

📖 **Full docs: [folk-project.github.io/folk-releases](https://folk-project.github.io/folk-releases/)**

| Guide | Link |
|-------|------|
| Installation (PIE, Docker, build from source) | [installation](https://folk-project.github.io/folk-releases/installation/) |
| Configuration (`folk.toml`) | [configuration](https://folk-project.github.io/folk-releases/configuration/) |
| Full config reference | [reference](https://folk-project.github.io/folk-releases/reference/) |
| Plugins — [HTTP](https://folk-project.github.io/folk-releases/plugins/http/) · [Jobs](https://folk-project.github.io/folk-releases/plugins/jobs/) · [gRPC](https://folk-project.github.io/folk-releases/plugins/grpc/) · [Metrics](https://folk-project.github.io/folk-releases/plugins/metrics/) · [Process](https://folk-project.github.io/folk-releases/plugins/process/) | [plugins](https://folk-project.github.io/folk-releases/) |
| Benchmarks | [benchmarks](https://folk-project.github.io/folk-releases/benchmarks/) |

## Quick Start

**1. Install the extension** (requires PHP 8.2+, NTS) — see [Installation](https://folk-project.github.io/folk-releases/installation/) for Docker and from-source:

```bash
pie install folk-project/ext-folk
```

**2. Install the SDK or a framework adapter:**

```bash
composer require folk/sdk        # plain PHP
# composer require folk/laravel  # Laravel
# composer require folk/spiral   # Spiral 3.x
# composer require folk/symfony  # Symfony 6.4 / 7.x / 8.x
# composer require folk/yii3     # Yii 3
```

**3. Create `folk.toml`** ([all options](https://folk-project.github.io/folk-releases/configuration/)):

```toml
[workers]
# Framework adapters (Laravel/Spiral/Symfony/Yii 3): vendor/bin/folk-server
# Plain SDK:                                          vendor/bin/folk-worker
script = "vendor/bin/folk-server"
count = 4

[http]
listen = "0.0.0.0:8080"
```

**4. Run:**

```bash
php vendor/bin/folk-server   # framework adapters; plain SDK: vendor/bin/folk-worker
```

Your app is now serving HTTP on port 8080 across 4 worker processes.

## Downloads

Pre-built extensions are available on the [Releases](https://github.com/Folk-Project/folk-releases/releases) page.

## Contributing

Found a bug or have an idea? See [Contributing Guide](https://folk-project.github.io/folk-releases/contributing/) for how to report issues and propose features.

All issues are tracked in this repository: [Issues](https://github.com/Folk-Project/folk-releases/issues)

## License

MIT
