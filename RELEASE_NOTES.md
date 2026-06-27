### What's new

Pluggable jobs drivers ([#46](https://github.com/Folk-Project/folk-releases/issues/46)).

The jobs plugin now hosts multiple queue backends behind one shared
consumer/retry/DLQ machinery. Each backend is a **connection** that nests its
own queues, and each backend is a Cargo feature — so a build only pulls in the
drivers it actually uses. This release ships the foundation plus three
simple-tier drivers: **memory**, **redis**, and a new pure-Rust persistent
**embedded** (redb) driver.

**New config layout.** A connection's key is the driver name, and it nests its
queues directly — no dangling references, no `default_connection`:

```toml
[jobs.connections.memory]
  [jobs.connections.memory.queues.default]
  concurrency = 4

[jobs.connections.redis]
host = "127.0.0.1"
port = 6379
tls = false
key_prefix = ""
  [jobs.connections.redis.queues.emails]
  concurrency = 8
  dead_letter_queue = "failed"

[jobs.connections.embedded]      # persistent, requires the "embedded" build feature
path = "var/jobs.redb"
durability = "eventual"
  [jobs.connections.embedded.queues.heavy]
  concurrency = 2
```

> **Breaking change.** The old flat `[jobs]` form (`driver`, `host`, `port`,
> `db`, `[[jobs.queues]]`) is removed. Move each queue under
> `[jobs.connections.<driver>.queues.<name>]`. One instance per driver type
> (two `[jobs.connections.redis]` sections is a TOML duplicate-key error).

**Addressing from PHP.** A job's queue name may carry an optional connection
prefix — `[<connection>.]<queue>` — so you can target a specific backend with a
plain `->onQueue("redis.emails")`. A bare name resolves directly when it is
unique across all connections; if the same name exists in more than one
connection it is ambiguous and `jobs.push` asks for a prefix. No new contract
field and no new `config/queue.php` key — the prefix rides inside the queue
string, and the backend stays invisible in job code.

**Feature-gated builds.** Default features are `["memory", "redis"]`; `embedded`
is opt-in. A connection section whose driver is not compiled in makes the server
**fail fast at startup** with an actionable message instead of a cryptic error.
Select drivers per build in `folk.build.toml`:

```toml
[[plugin]]
crate_name = "folk-plugin-jobs"
version = "0.4"
config_key = "jobs"
features = ["redis", "embedded"]
```

**folk-builder (0.2.8).** Two changes support the above:

* A new `features = [...]` field on `[[plugin]]` entries is forwarded to the
  generated Cargo dependency, so a build can opt into driver features.
* A new `folk-builder build --manifest build-manifest.toml` reads the release
  manifest **directly** — including the per-plugin `features` table form — and
  builds without an intermediate `folk.build.toml`. The release CI now uses this
  instead of a Python pre-processing step, so manifest parsing lives in one
  place (Rust) and is unit-tested. `--config folk.build.toml` still works.

**Embedded (redb) driver.** A pure-Rust, single-file ACID queue for persistent
jobs without an external broker — jobs survive a worker restart. Choose
`durability = "eventual"` (fast) or `"immediate"` (fsync per commit).

**PHP fixes.** Delayed dispatch now works end to end: Laravel
`Bus::dispatch()->delay(...)` (and `FolkQueue::later()`) sends the delay through
to the plugin; the Symfony/Spiral/Yii 3 `FolkQueue::push()` gained an
`int $delay = 0` parameter. Job handlers now receive the **real** queue the job
came from (the addressed `<connection>.<queue>`), instead of always seeing
`default`.

Native queue adapters for Symfony, Spiral and Yii 3 ([#76](https://github.com/Folk-Project/folk-releases/issues/76)).

Until now only Laravel plugged Folk into its framework's own queue; Symfony,
Spiral and Yii 3 had a bespoke `FolkQueue::push()` helper you called by hand.
This release makes Folk a **native** queue backend in all three, so apps
dispatch jobs the idiomatic way and Folk is "just another transport/driver".
No Rust changes — this builds entirely on the phase-74 `jobs.push`/`jobs.process`
contract; the bespoke `FolkQueue` helpers are kept but deprecated.

* **Symfony** — a Messenger transport. Register `FolkTransportFactory` (tag
  `messenger.transport_factory`) and route messages to a `folk://<connection>.<queue>`
  DSN; `MessageBusInterface::dispatch()` then flows through Folk and is consumed
  back into the bus by `FolkMessengerJobHandler`. `DelayStamp` maps to the
  plugin delay.

  ```yaml
  framework:
      messenger:
          transports:
              folk: 'folk://memory.default'
          routing:
              App\Message\SendEmail: folk
  ```

* **Spiral** — a `Spiral\Queue\QueueInterface` driver. Register it as a queue
  connection so `QueueInterface::push()` routes through Folk; consumption is
  resolved through Spiral's `HandlerRegistryInterface`.

  ```php
  // app/config/queue.php
  return [
      'default' => 'folk',
      'connections' => [
          'folk' => ['driver' => \Folk\Spiral\Jobs\FolkQueueDriver::class],
      ],
  ];
  ```

* **Yii 3** — a `yiisoft/queue` `AdapterInterface`. Bind `FolkQueueAdapter` as
  the queue adapter; `QueueInterface::push(new Message(...))` goes through Folk
  and is consumed through the Yii `Worker`. (`yiisoft/queue` has no stable tag
  yet — require `^3.0@dev`.)

All three keep the prefix addressing `[<connection>.]<queue>` and thread the
real queue into the handler. Job/message ids are now **UUID v7** (time-ordered,
consistent with Folk's `request_id`) via a new `Folk\Sdk\Uuid` helper.

### Versions

| Package | Version | Type |
|---------|---------|------|
| folk-api | 0.2.11 | crates.io |
| folk-core | 0.3.10 | crates.io |
| folk-ext | 0.3.10 | crates.io |
| folk-plugin-http | 0.4.4 | crates.io |
| folk-plugin-grpc | 0.2.7 | crates.io |
| folk-plugin-jobs | 0.4.0 | crates.io |
| folk-plugin-metrics | 0.2.3 | crates.io |
| folk-plugin-process | 0.2.4 | crates.io |
| folk-builder | 0.2.8 | crates.io |
| folk-sdk | 0.3.7 | packagist |
| folk/laravel | 0.3.8 | packagist |
| folk/spiral | 0.1.6 | packagist |
| folk/symfony | 0.1.6 | packagist |
| folk/yii3 | 0.1.5 | packagist |

> The PHP adapters now resolve `folk/sdk` from Packagist (the redundant `vcs`
> repository entry was dropped) — this avoids GitHub commits-API HTTP 429
> throttling during `composer install` in CI.
