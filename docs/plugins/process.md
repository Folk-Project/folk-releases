# Process Plugin

Managed background processes with automatic restart and supervision.

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
