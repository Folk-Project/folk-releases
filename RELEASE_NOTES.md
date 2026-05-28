### What's new

- **folk/sdk v0.2.4 + folk/laravel v0.2.3** — SDK refactor & dead code removal (phase 37):
  - Removed all legacy transport code: `Protocol/` namespace, `Rpc/RpcClient`, `ForkMasterLoop`, `runPipe()`, `runExtension()`
  - `WorkerLoop::run()` simplified to direct `folk_worker_run()` call only
  - `GrpcRouter` moved from folk-laravel to `Folk\Sdk\Grpc\GrpcRouter` (framework-agnostic)
  - New `ResettableInterface` — typed contract for request resetters
  - `HandlerLoop::registerResetter()` now requires `ResettableInterface` instead of `object`
  - New `PsrHttpHandler` abstract base for PSR-7 frameworks (Spiral, Symfony, Yii3)
  - `bin/folk-worker` replaced with framework-agnostic entry point
  - folk-laravel: all 4 resetters implement `ResettableInterface`, fork-mode hook removed
  - folk-core (Rust): removed 3 dead crates (`folk-protocol`, `folk-runtime-pipe`, `folk-runtime-fork`), deprecated `folk` binary, orphaned `rpc_registry.rs`/`rpc_server.rs`

### Versions

| Package | Version | Type |
|---------|---------|------|
| folk/sdk | 0.2.4 | packagist |
| folk/laravel | 0.2.3 | packagist |
| folk-core | 0.2.5 | crates.io |
| folk-ext | 0.2.5 | crates.io |
| folk-plugin-http | 0.2.2 | crates.io |
| folk-plugin-jobs | 0.3.0 | crates.io |
| folk-plugin-grpc | 0.2.2 | crates.io |
| folk-plugin-metrics | 0.2.1 | crates.io |
| folk-plugin-process | 0.2.1 | crates.io |
