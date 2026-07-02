### 0.2.1 — micro-fix

- **`pie install` on NTS PHP.** The phase-79 prebuilt binaries are Non-Thread-Safe
  (NTS), but the extension package still declared `support-nts: false`, so
  `pie install folk-project/ext-folk` aborted on NTS PHP with
  *IncompatibleThreadSafetyMode*. Metadata now correctly declares NTS support —
  no binary change, just packaging.

### What's new

Runtime model: fork-after-warm worker processes (threads → processes)
([#1](https://github.com/Folk-Project/folk-releases/issues/1),
[#3](https://github.com/Folk-Project/folk-releases/issues/3)).

Folk no longer runs its workers as threads inside one PHP process. A
single-threaded **master** boots PHP and your framework once, then `fork()`s N
**worker processes**; each worker runs its own event loop and handles requests
in its own process, while the master only supervises. This is the model Swoole
and php-fpm use, and it unlocks three things the thread model never could:

- **Crash isolation.** A segfault or fatal in one worker no longer takes down
  the whole server — the master respawns it (typically < 0.5 s) while the other
  workers keep serving.
- **Force-kill of a stuck request (`exec_timeout` is now a HARD deadline).** A
  per-worker watchdog kills a worker whose request overruns `exec_timeout` and
  the master respawns it. (Previously the deadline was soft — a wedged PHP thread
  could not be reclaimed.)
- **Per-worker memory recycling (`max_memory_mb`).** The master samples each
  worker's RSS and gracefully recycles any that grows past the limit.

Connections are spread across workers with `SO_REUSEPORT` (HTTP and gRPC); jobs
become **competing consumers** (each worker is its own broker consumer — exactly
the right model for RabbitMQ/SQS/NATS/beanstalk/Kafka/Pub-Sub). Metrics from all
workers are aggregated through a shared-memory segment, so `/metrics` reports
process-wide totals. Dev hot-reload now re-execs the master for a clean
bootstrap on file change.

**Breaking changes**

- **PHP is now NTS** (non-thread-safe): base images move from `php:8.x-zts` to
  `php:8.x`. ZTS is no longer required (or used).
- **Plugin API (`folk-api` 0.3).** Plugins now declare where they run
  (`Placement::WorkerOnly` / `MasterOnly` / `Both`) and receive a `PluginRole`.
  All first-party plugins are updated; third-party plugins must migrate.
- **Adapter entry point.** `bin/folk-server` now bootstraps the framework once
  and calls `Folk\Server::serveForked()` instead of `start()` + the worker loop.
  `folk_is_worker_thread()` is obsolete (there are no worker threads). **Your
  application code is unchanged** — only the adapter's entry plumbing.

**New config** (see the reference): `[workers] max_memory_mb`, `[metrics]
max_series`; `[workers] exec_timeout` is now a hard deadline. Optional
`opcache.preload` (via `php -d` / php.ini) shares compiled framework opcodes
across workers from the master's warm interpreter.

Validated end-to-end on Laravel, Symfony, Spiral and Yii 3.
