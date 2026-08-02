# Release Notes

### `on_init` — pre-start lifecycle hooks

Folk can now run setup commands **itself** before the framework boots and any
listener binds — so an "everything in one container" deployment no longer needs a
separate `entrypoint.sh`. Configure them in `folk.toml`:

```toml
[on_init]
exec_timeout = "60s"       # default per-step timeout
exit_on_error = true       # non-zero exit / timeout aborts startup (fail-fast)
commands = [
  "cp -n .env.example .env",
  "composer install --no-dev --optimize-autoloader",
  { run = "php artisan migrate --force", exec_timeout = "300s" },
]
```

**What it does.** Each `commands` entry is either a bare shell string (uses the
section defaults) or an inline table overriding `run` / `exec_timeout` / `env` /
`user` / `exit_on_error`. Steps run **sequentially**, in order, from the project
root, via `sh -c` (pipes and `&&` work). Command output is streamed into the Folk
log.

**Fail-fast by default.** Unlike RoadRunner's `on_init` (log-and-continue), Folk
defaults `exit_on_error = true`: a non-zero exit or a per-step timeout aborts
startup, so a broken environment (a failed migration, a missing dependency) never
receives traffic. Opt out globally or per step. A step that overruns its
`exec_timeout` is killed (`SIGTERM` → grace → `SIGKILL` of its process group).

**Readiness.** While `on_init` runs, no listener is bound — an orchestrator's
readiness probe sees connection-refused (= not ready), and a fail-fast abort exits
non-zero so the container restarts instead of serving a half-set-up app.

**When it runs.** At the very top of the entry script, *before*
`vendor/autoload.php` is required, so even pre-bootstrap commands (a fresh `.env`,
a dependency refresh) take effect. Two caveats: it needs a shell (`sh -c`) — a
distroless image without one can't use string commands — and it is not a
from-scratch installer (the launcher lives in `vendor/bin`, so `composer install`
here refreshes an existing `vendor/`, it can't create an empty one). Absent
`[on_init]` (or empty `commands`) is a no-op: startup is unchanged.

**Availability.** Works across all four framework adapters (Laravel, Symfony,
Spiral, Yii 3). Requires the rebuilt extension.

**Versions:** folk-core / folk-ext **0.6.4** (config section + `on_init` engine +
native `folk_on_init()`), prebuilt extension rebuilt. The framework adapters ship a
one-line entry-script change (`bin/folk-server`) and a stub update. folk-api, the
plugins, and folk-builder are untouched.
