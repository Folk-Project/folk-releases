### 0.2.6 — auth/session leak + multi Set-Cookie ([#86](https://github.com/Folk-Project/folk-releases/issues/86))

**P0: authenticated state leaked between requests on a warm worker, and cookie
handling was broken on the persistent-worker HTTP path.** Fixing #86 uncovered
four related defects, all fixed here. Session-based auth (Laravel/Symfony/Spiral/
Yii3) now works correctly on Folk and no longer leaks between clients.

**1 — Auth/session no longer leaks (Laravel).** A request carrying no session
cookie could be served as the previously logged-in user. `SessionResetter` now
flushes the cached session store between requests, and `AuthResetter` calls
`AuthManager::forgetGuards()` (was per-guard `forgetUser()`, which left
`loggedOut`/`recallAttempted` set and ignored custom guards).

**2 — Request cookies are now parsed.** The adapters never fed the incoming
`Cookie` header to the framework request (`SymfonyRequest::create()` and PSR-7
don't derive it), so every request got a fresh, empty session — sessions, auth
and CSRF silently didn't work on a persistent worker. A shared
`Folk\Sdk\Http\CookieParser` now wires request cookies into Laravel, Symfony,
Spiral and Yii3.

**3 — Multiple `Set-Cookie` headers.** Several cookies (e.g. `XSRF-TOKEN` +
`laravel_session`) were folded into one comma-joined header and corrupted.
Responses now emit one `Set-Cookie` per cookie on both the buffered and the
streamed (`Folk::writeHead`) paths. `ResponseChunk::Headers` carries ordered
`(name, value)` pairs (`folk-api` 0.3.1) instead of a map.

**4 — Resetters run after a failed request too.** `WorkerLoop::dispatchDirect`
now runs per-request resetters in a `finally`, so a request that throws mid-way
can't leak its state into the next one.

**Versions.** `folk-api` 0.3.1, `folk-core`/`folk-ext` 0.5.1, `folk-plugin-http`
0.5.2, `folk-builder` 0.2.15; `folk/sdk` 0.4.1, `folk/laravel` 0.4.2,
`folk/symfony` · `folk/spiral` · `folk/yii3` 0.2.1. Prebuilt `.so` rebuilt
(build-manifest 0.2.6). No plugin cascade — `folk-plugin-jobs`/`grpc`/`metrics`/
`process` are unchanged.
