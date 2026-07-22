# Process Plugin

Managed background processes with automatic restart and supervision.

## Features

- Start, monitor, and restart background processes
- 3 restart policies: `always`, `on_failure`, `never`
- Max restart limit with configurable delay
- Environment variables per process
- Working directory per process
- stdout/stderr capture: inherit, file, or null
- Configurable stop timeout and stop signal (TERM, INT, QUIT)
- Multiple process instances via `numprocs`
- Shell-aware command parsing (quoted arguments supported)
- Graceful shutdown (configurable signal + timeout)
- Health check integration — process status visible on `/health`
- 5 Prometheus metrics (`folk_process_*`)
- RPC: `process.list`, `process.restart`

## Configuration

```toml
[[process.processes]]
name = "scheduler"
command = "php artisan schedule:work"
restart = "always"              # "always", "on_failure", or "never"
max_restarts = 5                # Max restart attempts
restart_delay = "2s"            # Delay between restarts
directory = "/app"              # Working directory (default: server CWD)
stop_timeout = "10s"            # Graceful shutdown timeout (default: 5s)
stop_signal = "TERM"            # Stop signal: TERM, INT, QUIT (default: TERM)
numprocs = 1                    # Number of process copies (default: 1)

[process.processes.env]         # Environment variables
APP_ENV = "production"
QUEUE_CONNECTION = "redis"

[process.processes.logging]
stdout = "inherit"              # inherit | null | { file = "/path" }
stderr = "inherit"

[[process.processes]]
name = "queue-worker"
command = "php artisan queue:work --tries=3"
restart = "on_failure"
max_restarts = 10
restart_delay = "5s"
numprocs = 4                    # Starts queue-worker:0 .. queue-worker:3

[process.processes.logging]
stdout = { file = "/var/log/folk/queue.log" }
stderr = { file = "/var/log/folk/queue-err.log" }
```

## Restart Policies

| Policy | Behavior |
|--------|----------|
| `always` | Restart regardless of exit code |
| `on_failure` | Restart only on non-zero exit code |
| `never` | Never restart |

After `max_restarts` is reached, the process is left stopped and a warning is logged.

## Stop Signals

| Signal | Description |
|--------|-------------|
| `TERM` | Default. Graceful termination |
| `INT` | Interrupt (like Ctrl+C) |
| `QUIT` | Quit with core dump |

## Logging

| Target | Behavior |
|--------|----------|
| `inherit` | Output goes to the Folk server log (default) |
| `null` | Output discarded |
| `{ file = "/path" }` | Append to file |

## Metrics

| Metric | Type | Labels | Description |
|--------|------|--------|-------------|
| `folk_process_up` | Gauge | `name` | 1 if running, 0 if not |
| `folk_process_restarts_total` | Counter | `name` | Total restart count |
| `folk_process_uptime_seconds` | Gauge | `name` | Current instance uptime |
| `folk_process_exit_code` | Gauge | `name` | Last exit code |
| `folk_process_status` | Gauge | `name`, `status` | 1 for active status (running/stopped/failed) |

## RPC Methods

| Method | Payload | Description |
|--------|---------|-------------|
| `process.list` | none | List all supervised process statuses |
| `process.restart` | `"process-name"` (JSON string) | Restart a named process |
