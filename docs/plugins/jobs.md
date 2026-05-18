# Jobs Plugin

Background job processing with in-memory or Redis-backed queues.

## Configuration

```toml
[jobs]
driver = "memory"                          # "memory" or "redis"
redis_url = "redis://127.0.0.1:6379"       # Redis URL (when driver = "redis")

[[jobs.queues]]
name = "default"
concurrency = 4        # Concurrent workers for this queue
max_retries = 3        # Max retry attempts

[[jobs.queues]]
name = "priority"
concurrency = 8
max_retries = 5
```

## Drivers

| Driver | Persistence | Use case |
|--------|-------------|----------|
| `memory` | No — lost on restart | Development, testing |
| `redis` | Yes | Production |

## PHP Usage

Dispatch a job:

```php
folk_call('jobs.dispatch', json_encode([
    'queue' => 'default',
    'handler' => 'App\\Jobs\\SendEmail',
    'payload' => ['to' => 'user@example.com'],
]));
```

With Laravel, use the standard `dispatch()` helper — the Folk service provider routes jobs through the jobs plugin automatically.
