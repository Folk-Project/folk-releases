# Process Plugin

Managed background processes with automatic restart and supervision.

## Features

- Start, monitor, and restart background processes
- 3 restart policies: `always`, `on_failure`, `never`
- Max restart limit with configurable delay
- Graceful shutdown (SIGTERM + timeout)
- Health check integration — process status visible on `/health`
- RPC: `process.list` (supervised process statuses)

## Planned

- Environment variables per process
- Working directory
- stdout/stderr capture and logging
- Configurable stop timeout (currently 5s hardcoded)
- Multiple process instances (numprocs)
- Custom stop signal (TERM, INT, QUIT)
- `process.restart` RPC

## Configuration

```toml
[[process.processes]]
name = "scheduler"              # Process name (for logs)
command = "php artisan schedule:work"  # Shell command
restart = "always"              # "always", "on_failure", or "never"
max_restarts = 5                # Max restart attempts
restart_delay = "2s"            # Delay between restarts

[[process.processes]]
name = "queue-worker"
command = "php artisan queue:work --tries=3"
restart = "on_failure"
max_restarts = 10
restart_delay = "5s"
```

## Restart Policies

| Policy | Behavior |
|--------|----------|
| `always` | Restart regardless of exit code |
| `on_failure` | Restart only on non-zero exit code |
| `never` | Never restart |

After `max_restarts` is reached, the process is left stopped and a warning is logged.
