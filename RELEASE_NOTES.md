### 0.2.2 — static file serving + persistent-worker state reset

Two P0 fixes for apps served through Folk.

**Static files from `public/` ([#82](https://github.com/Folk-Project/folk-releases/issues/82)) — `folk-plugin-http` 0.5.1.**
Folk dispatched every request to PHP, so built assets (`/build/assets/*.css`,
`*.js`, images, fonts) fell through to framework routing and returned a 404 —
a Vite/Inertia frontend rendered unstyled. New opt-in `[http] public_dir`
serves matching files straight from disk (nginx `try_files` style) before
dispatching to PHP; a miss falls through to the framework. `.php` files and
non-`GET`/`HEAD` requests always go to PHP, so the front controller is never
returned as source. Disabled by default.

```toml
[http]
public_dir = "public"   # serve public/ from disk, else dispatch to PHP
```

**Persistent worker replayed the first Inertia response ([#83](https://github.com/Folk-Project/folk-releases/issues/83)) — `folk/laravel` 0.4.1.**
Because Folk keeps the app booted across requests (Octane-style), the
container's `scoped` instances were never flushed between requests. Inertia's
`SsrState` is a scoped binding that caches the rendered SSR response, so the
first request's props/HTML were replayed for every later request for the
worker's lifetime (plain routes were unaffected). Folk now flushes scoped
instances and Inertia's shared props between requests, matching Octane:

- **`ScopedResetter`** — `forgetScopedInstances()` per request.
- **`InertiaResetter`** — `Inertia::flushShared()` (no-op when Inertia is absent).
- **`config('folk.resetters')`** — register your own per-request resetters
  (any `ResettableInterface`); a hook for packages/apps that keep request-scoped
  state.

No application code changes required.

**Prebuilt extension.** The `0.2.2` prebuilt `folk.so` is rebuilt with
`folk-plugin-http` 0.5.1 (adds `public_dir`); no other plugin changed.
