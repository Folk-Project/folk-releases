# Jobs Plugin

Background job processing with in-memory or Redis-backed queues.

## Features

- Three drivers: **memory**, **Redis**, and **embedded** (persistent, pure-Rust redb)
- Feature-gated builds — a build only pulls in the drivers it uses
- Multiple named queues with independent concurrency
- Configurable retry: delay, backoff strategy (exponential / linear / fixed)
- Job execution timeout
- Dead letter queue for failed jobs
- Delayed jobs (scheduled for future execution)
- Priority queues
- RPC: `jobs.push` (add job with optional delay), `jobs.stats` (queue depth)
- Graceful shutdown — in-flight jobs complete before exit
- Prometheus metrics: pushed, processed, duration, retries, DLQ, active

## Configuration

Each backend (driver) is declared as a **connection** that nests its own
queues. The connection key **is** the driver name, so there is one instance per
driver type (two `[jobs.connections.redis]` sections is a TOML duplicate-key
error). Different drivers can run side by side.

```toml
# In-memory driver — no persistence. The section just declares queues.
[jobs.connections.memory]
  [jobs.connections.memory.queues.default]
  concurrency = 4                  # Concurrent consumers for this queue
  max_retries = 3                  # Retries before DLQ or discard
  retry_delay = "1s"               # Base delay between retries
  retry_backoff = "exponential"    # "exponential", "linear", or "fixed"
  job_timeout = "60s"              # Max execution time per job ("0s" = unlimited)
  dead_letter_queue = "failed"     # Queue for failed jobs (omit to discard)
  priority = 10                    # Lower = higher priority

# Redis driver (production).
[jobs.connections.redis]
host = "127.0.0.1"
port = 6379
username = ""                      # Redis ACL username (optional)
password = ""
db = 0
tls = false                        # true → rediss:// scheme
key_prefix = ""                    # Prefix applied to all queue keys
pool_size = 8
connect_timeout = "5s"
command_timeout = "5s"
url = ""                           # Full URL override; non-empty wins over the fields above
  [jobs.connections.redis.queues.emails]
  concurrency = 8
  dead_letter_queue = "failed"

# Embedded driver (redb) — pure-Rust persistent ACID queue, single file.
# Requires the "embedded" build feature.
[jobs.connections.embedded]
path = "var/jobs.redb"
durability = "eventual"            # "eventual" (fast) or "immediate" (fsync per commit)
  [jobs.connections.embedded.queues.heavy]
  concurrency = 2
```

> **Migrated from a flat `[jobs]`?** The old form (`driver`, `host`, `port`,
> `db`, `[[jobs.queues]]`) was removed — move each queue under
> `[jobs.connections.<driver>.queues.<name>]`.

### Queue options reference

A queue's name is its map key (`...queues.<name>`), not a field.

| Option | Type | Default | Description |
|--------|------|---------|-------------|
| `concurrency` | int | `4` | Number of concurrent consumers |
| `max_retries` | int | `3` | Retries before DLQ or discard |
| `retry_delay` | duration | `"1s"` | Base delay between retries |
| `retry_backoff` | string | `"exponential"` | Backoff: `exponential`, `linear`, `fixed` |
| `job_timeout` | duration | `"60s"` | Execution timeout (`"0s"` = unlimited) |
| `dead_letter_queue` | string | — | Queue name for failed jobs |
| `priority` | int | `10` | Lower = processed first |

### Backoff strategies

| Strategy | Delay formula | Example (base=1s) |
|----------|---------------|--------------------|
| `exponential` | base × 2^attempt | 1s, 2s, 4s, 8s, 16s |
| `linear` | base × (attempt+1) | 1s, 2s, 3s, 4s, 5s |
| `fixed` | base | 1s, 1s, 1s, 1s, 1s |

### Redis connection fields

| Field | Default | Meaning |
|-------|---------|---------|
| `host` / `port` | `127.0.0.1` / `6379` | Redis endpoint |
| `username` / `password` | `""` | ACL username / AUTH password |
| `db` | `0` | Database number (0–15) |
| `tls` | `false` | `true` → `rediss://` |
| `key_prefix` | `""` | Prefix for all queue keys |
| `pool_size` | `8` | Connection pool size |
| `connect_timeout` / `command_timeout` | `"5s"` | Timeouts |
| `url` | `""` | Full URL override (wins over the fields above) |

The `memory` connection takes no fields (queues only); the `embedded`
connection takes `path` and `durability`.

## Drivers

| Driver | Build feature | Persistence | Use case |
|--------|---------------|-------------|----------|
| `memory` | always on | No — lost on restart | Development, testing |
| `redis` | `redis` (default) | Yes | Production |
| `embedded` (redb) | `embedded` (opt-in) | Yes — single file, ACID | Persistent jobs without an external broker |

Drivers are compiled into the build via `folk.build.toml` (default features are
`["memory", "redis"]`; add `"embedded"` to `features`). A connection whose
driver is not compiled in makes the server **fail fast at startup** with an
actionable message.

> Broker-backed drivers (RabbitMQ, Kafka, NATS, SQS, Pub/Sub, Beanstalk) are on
> the roadmap and not yet shipped.

## Delayed jobs

Push a job with a delay (seconds):

```php
folk_call('jobs.push', json_encode([
    'queue' => 'default',
    'payload' => $serializedJob,
    'delay' => 30,  // execute after 30 seconds
]));
```

Redis implementation uses ZADD sorted set with a polling promoter (1s interval).

## Metrics

The jobs plugin registers the following Prometheus metrics (via the metrics plugin):

| Metric | Type | Labels | Description |
|--------|------|--------|-------------|
| `folk_jobs_pushed_total` | Counter | `queue` | Jobs added to queue |
| `folk_jobs_processed_total` | Counter | `queue`, `status` | Jobs processed (ok/failed) |
| `folk_jobs_processing_duration_seconds` | Histogram | `queue` | Processing time |
| `folk_jobs_retries_total` | Counter | `queue` | Retry attempts |
| `folk_jobs_dead_letter_total` | Counter | `queue` | Jobs sent to DLQ |
| `folk_jobs_active` | Gauge | `queue` | Jobs being processed now |

## PHP Usage

Each framework adapter plugs Folk into the framework's **native** queue
abstraction, so you dispatch jobs the idiomatic way and Folk is "just another
transport/driver". The queue name may carry an optional connection prefix —
`[<connection>.]<queue>` — to target a specific backend; the backend stays
invisible in job code.

### Laravel

The service provider registers a `folk` queue connection. Use the standard
`dispatch()` helper:

```php
TestJob::dispatch('hello');                  // default queue
TestJob::dispatch('hello')->onQueue('redis.emails');
TestJob::dispatch('hello')->delay(30);       // delayed
```

Requires `QUEUE_CONNECTION=folk` in `.env`.

### Symfony

Folk is a Messenger **transport**. Register the factory and route messages to a
`folk://<connection>.<queue>` DSN:

```yaml
# config/services.yaml
services:
    Folk\Symfony\Jobs\FolkTransportFactory:
        tags: ['messenger.transport_factory']

    Folk\Symfony\Jobs\FolkMessengerJobHandler:
        public: true
        arguments:
            $bus: '@messenger.bus.default'
            $serializer: '@messenger.transport.native_php_serializer'
```

```yaml
# config/packages/messenger.yaml
framework:
    messenger:
        transports:
            folk: 'folk://memory.default'
        routing:
            App\Message\SendEmail: folk
```

```php
$bus->dispatch(new SendEmail(...));                       // routed to Folk
$bus->dispatch(new SendEmail(...), [new DelayStamp(30000)]); // 30s delay
```

Consumption is driven by Folk (no `messenger:consume` loop): the worker hands
the message back into the bus, where your `#[AsMessageHandler]` runs.

### Spiral

Folk is a `Spiral\Queue\QueueInterface` driver. Register it as a queue
connection (a bootloader binding is overridden by Spiral's queue injector):

```php
// app/config/queue.php
use Folk\Spiral\Jobs\FolkQueueDriver;

return [
    'default' => 'folk',
    'connections' => [
        'folk' => ['driver' => FolkQueueDriver::class],
    ],
];
```

```php
use Spiral\Queue\Options;
use Spiral\Queue\QueueInterface;

$queue->push(SendEmail::class, ['to' => $addr], Options::onQueue('redis.emails')->withDelay(30));
```

Consumption resolves the handler through Spiral's `HandlerRegistryInterface`
(`HandlerInterface::handle($name, $id, $payload)`).

### Yii 3

Folk is a `yiisoft/queue` `AdapterInterface`. Bind it as the queue adapter and
map the message type to a handler:

```php
// config/common/di/queue.php
use Folk\Yii3\Jobs\FolkQueueAdapter;
use Yiisoft\Definitions\Reference;
use Yiisoft\Queue\Adapter\AdapterInterface;
use Yiisoft\Queue\Message\Serializer\MessageSerializerInterface;

return [
    AdapterInterface::class => [
        'class' => FolkQueueAdapter::class,
        '__construct()' => [
            'serializer' => Reference::to(MessageSerializerInterface::class),
            'channel' => 'memory.default',
        ],
    ],
];
```

```php
// config/common/params.php — map message type to handler
'yiisoft/queue' => ['handlers' => ['app.send-email' => SendEmailHandler::class]],
```

```php
use Yiisoft\Queue\Message\GenericMessage;

$queue->push(GenericMessage::fromPayload('app.send-email', ['to' => $addr]));
```

`yiisoft/queue` has no stable release yet — require `^3.0@dev`.

### Low-level helper

All three non-Laravel adapters still ship a bespoke `FolkQueue::push()` helper
for apps without the framework's queue package, but it is **deprecated** in
favour of the native paths above. Job/message ids are generated as UUID v7 via
`Folk\Sdk\Uuid`.
