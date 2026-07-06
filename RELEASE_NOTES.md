### 0.2.4 — worker process isolation

Thanks to [@Ferikl](https://github.com/Ferikl) for reporting the process-leak
problem that motivated this release.

**Worker process-group isolation ([#84](https://github.com/Folk-Project/folk-releases/issues/84)) — `folk-core`/`folk-ext` 0.4.1.**
Fork-after-warm workers are long-lived. A **live detached process** spawned from
a request — `exec("cmd &")`, a self-daemonizing `proc_open`, `nohup` — used to
outlive its worker: it survived recycle/force-kill, was re-parented to the master
and **accumulated** until the pod ran out of resources (the same class of problem
FrankenPHP hit). Reaping never touched it because it was alive, not a zombie.

Each worker (and the master-services process) is now the leader of its **own
process group** (`setpgid`). Any process it spawns inherits that group, so a
**force-kill takes the whole subtree with it** (`kill(-pgid)`):

- **exec_timeout watchdog** — a wedged worker is killed together with its subtree.
- **shutdown drain** — the final `SIGKILL` targets the group, not just the worker.
- **Graceful RSS recycle** still signals only the worker (`SIGTERM`), so in-flight
  synchronous `exec` children finish normally.

**`destroy_timeout` escalation ([#85](https://github.com/Folk-Project/folk-releases/issues/85)).**
A worker recycled for memory that **ignores `SIGTERM`** (wedged in a C call) used
to hang in the recycling state forever, silently shrinking the pool. A new
`[workers] destroy_timeout` (default `10s`) bounds this: if the worker hasn't
exited within the window, the master escalates to a group `SIGKILL`.

```toml
[workers]
max_memory_mb = 256
destroy_timeout = "10s"   # SIGTERM → wait → SIGKILL the worker's process group
```

**`pcntl_fork()` from a request is not supported — now guarded.**
Forking a request duplicates the worker's already-multithreaded runtime (tokio +
watchdog); the child inherits broken mutexes and would hang (S-state, leaking
PIDs). Folk now ports FrankenPHP's fix: the fork child is `_exit`-ed immediately
after the PHP call instead of re-entering the runtime, and the worker reaps the
resulting short-lived child so it does not accumulate. **For background or
parallel work use `folk-plugin-jobs` or `folk-plugin-process`, not `exec("&")`
or `pcntl_fork()`.** See *Configuration → Background processes*.

**Scope.** `folk-core`/`folk-ext` only — `folk-api`, the plugins and the PHP
contract are unchanged. Docker-smoked: detached `exec` subtree dies on force-kill
(with a negative control proving the leak before the fix), `pcntl_fork` no longer
hangs or accumulates, `destroy_timeout` escalation fires, and synchronous `exec`,
crash isolation, graceful recycle and `exec_timeout` all still work.

**Prebuilt extension.** The `0.2.4` prebuilt `folk.so` is rebuilt against
`folk-ext` 0.4.1; no plugin crate changed.
