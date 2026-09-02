# Release Notes

### Dev hot-reload no longer hangs

`[dev] watch = true` reloads reliably again. Previously, on a real app, editing a
watched file could leave the server **up but unresponsive** — every request got
`connection refused` / `502` with no recovery until a manual restart.

**What was wrong.** Fork-after-warm hot reload works by having the master drain its
workers and re-exec itself for a fresh, fully re-warmed bootstrap (a plain re-fork
can't pick up changed code — forked workers inherit the master's warm image via
copy-on-write). The drain's final step waited for **all** children to exit with an
unbounded blocking reap. In a container the master runs as PID 1, so it also reaps
orphans: any process your app detached into its own session (a scheduler tick, a
queue worker, a sidecar) re-parents to the master when its worker dies, and the
master then blocked on it forever — the re-exec never ran, the workers stayed dead,
the port went silent.

**The fix.** The drain now reaps with a bounded, non-blocking loop: killed workers
release their `SO_REUSEPORT` sockets immediately, so the fresh master rebinds and
serves even if a detached grandchild is still lingering. Hot reload recovers in a
fraction of a second instead of hanging.

Also cleaned up: in the fork model the legacy in-process file watcher is inert (each
worker runs a single, non-recyclable main thread), so it no longer starts in workers
and the misleading `Set workers.count > 1` warning is gone — reload is driven by the
master. Production (no `[dev] watch`) is unaffected.

**Upgrade.** Rebuilt `folk.so` (folk-core / folk-ext `0.6.5`). No config changes.
