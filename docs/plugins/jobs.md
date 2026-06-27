# Jobs Plugin

Background job processing with in-memory or Redis-backed queues.

## Features

- In-memory and Redis drivers
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

```toml
[jobs]
driver = "memory"       # "memory" or "redis"

# Redis connection (used when driver = "redis")
host = "127.0.0.1"
port = 6379
password = ""
db = 0

[[jobs.queues]]
name = "default"
concurrency = 4                    # Concurrent workers for this queue
max_retries = 3                    # Max retry attempts before DLQ/discard
retry_delay = "1s"                 # Base delay between retries
retry_backoff = "exponential"      # "exponential", "linear", or "fixed"
job_timeout = "60s"                # Max execution time per job ("0s" = no limit)
dead_letter_queue = "failed"       # Queue for failed jobs (omit to discard)
priority = 10                      # Lower = higher priority

[[jobs.queues]]
name = "critical"
concurrency = 8
max_retries = 5
retry_delay = "500ms"
retry_backoff = "exponential"
job_timeout = "30s"
dead_letter_queue = "failed"
priority = 1
```

### Queue options reference

| Option | Type | Default | Description |
|--------|------|---------|-------------|
| `name` | string | `"default"` | Queue name |
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

### Connection config

| Field | Default | Redis meaning |
|-------|---------|---------------|
| `host` | `"127.0.0.1"` | Redis host |
| `port` | `6379` | Redis port |
| `password` | `""` | AUTH password |
| `db` | `0` | Database number (0-15) |

## Drivers

| Driver | Persistence | Use case |
|--------|-------------|----------|
| `memory` | No — lost on restart | Development, testing |
| `redis` | Yes | Production |

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

### Laravel (`folk/laravel`)

The service provider registers a `folk` queue connection. Use the standard
`dispatch()` helper:

```php
TestJob::dispatch('hello');                  // default queue
TestJob::dispatch('hello')->onQueue('redis.emails');
TestJob::dispatch('hello')->delay(30);       // requires folk/laravel >= 0.3.0
```

Requires `QUEUE_CONNECTION=folk` in `.env`.

### Symfony (`folk/symfony` >= 0.1.5)

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

### Spiral (`folk/spiral` >= 0.1.5)

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

### Yii 3 (`folk/yii3` >= 0.1.4)

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
