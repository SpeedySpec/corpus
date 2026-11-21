# ADR-025: PHP runs in supervised worker pools, one request per worker

**Status:** Accepted, superseded in part by the PHP runtime and admin gate design (where the pools run)
**Date:** 2026-11-18
**Deciders:** the designer
**Exploration:** [PHP foundation](../speedyspec-server/explorations/2026-11-18-php-foundation.md)
**Related:** [ADR-016](016-rust-runs-as-an-embedded-php-extension.md), [ADR-022](022-fastcgi-over-a-unix-socket.md), [ADR-024](024-static-php-with-every-wordpress-extension.md), [ADR-026](026-a-writable-site-directory.md)

---

## Context

Before this decision, the server's PHP mounts answered 503: nothing could run the front-end API or the blog. PHP
workers crash, hang and leak memory, and WordPress expects every request to start from a clean state. Releases are
switched atomically, so a new release's PHP must be ready before it goes live, and the old one must finish its
requests.

### Why this matters

A worker that crashes or leaks could take requests down with it, or poison the next request. A request sent to a
worker already known to be broken would fail for no reason. A worker mode that keeps state between requests would
break WordPress's assumptions about a fresh start.

---

## Decision

PHP apps run in supervised pools of worker processes, one request per worker at a time, like php-fpm's `dynamic`
mode. Broken workers are killed and replaced at once. Since the PHP runtime design, the pools run in their own binary,
`speedyspec-php-runtime`, not in the server.

### Rules

1. One request per worker at a time. Each request runs a full PHP request startup and shutdown, with `$_SERVER` from
   the FastCGI parameters and the body on `php://input`. The response is streamed.
2. Every request starts clean. There is no worker mode that keeps state between requests.
3. Pool settings per app: `workers` (min, max), `max_requests`, `request_timeout`, `memory_limit`, `idle_timeout`.
4. A worker is killed and replaced when it exits or dies (the request gets 502), runs past `request_timeout`
   (SIGKILL, 504), passes `memory_limit` after a request, has served `max_requests`, or the engine reports it poisoned.
5. Each kill or recycle is logged, and a replacement starts at once.
6. `prepare` starts a release's pool and waits for its first worker to answer; `activate` swaps to the new pool in
   one step; `retire` drains the old pool.
7. A release declares `document_root`, the `entry` front controller, and per-app `ini` values.
8. The server binary holds no PHP. The pools run in `speedyspec-php-runtime`, released and upgraded like any app.

### What this looks like

✅ **Correct**

```toml
# Per-app pool settings (defaults shown).
[app.blog.php]
workers = { min = 1, max = 4 }
max_requests = 1000
request_timeout = "60s"
memory_limit = "256MiB"
idle_timeout = "10s"
```

❌ **Not permitted**

```php
// A persistent worker loop that keeps state between requests (FrankenPHP-style; a made-up API).
while ( speedyspec_handle_request( function () use ( $app ) { $app->run(); } ) ) {
	// $app and its globals survive into the next request
}
```

### Patterns

- **Object Pool:** the worker pool, with supervision that kills and replaces bad workers.

---

## Consequences

### Positive

1. A crash, hang or leak costs one request at most, and never reaches the next one.
2. WordPress gets the clean start it expects.
3. Release switches happen without failed requests: the new pool is ready before it goes live.
4. Since the runtime became its own binary, PHP changes need no server release.

### Negative

1. A full PHP startup and shutdown per request costs more than a persistent worker.
   **Mitigation:** none; a clean start is required.
2. Pools belong to a server generation, so a handoff must start the new generation's pools before reporting ready.
   **Mitigation:** the handoff does this.
3. A separate runtime binary needs a versioned contract with the server.
   **Mitigation:** both sides refuse an unknown contract major, naming both versions.

### Trade-offs

**We prioritise:**
- Isolation and a clean start per request.
- Release switches with no failed requests.

**We accept:**
- The cost of a full request cycle per request.
- A second binary and a contract between the two.

---

## Fleet impact

- **speedyspec-server:** supervises pools through the runtime and holds no PHP. It lost its `php-worker` role.
- **speedyspec-php-runtime:** runs the workers, one pool per PHP app, and reports memory, request count and poisoning
  back to the server.
- **PHP apps (admin API, front-end API, blog):** must declare `document_root`, `entry` and `ini` in their manifest,
  and must not rely on state surviving between requests. Whether their releases use the new manifest fields yet is
  unverified.
- **speedyspec-cli:** not affected.

---

## Enforcement

### Automated

- `cargo test -p speedyspec-server --features test-hooks --test php` covers the pool: `crash_is_502_and_worker_replaced`,
  `hang_is_504_and_worker_killed`, `leak_recycles_worker`, `max_requests_recycles_worker`,
  `grows_to_max_and_shrinks_after_idle` and `release_switch_finishes_long_request_on_old_pool`. A failure fails
  `cargo test`. It runs in the server's CI `php` job. That step has failed in CI before; no failure has been traced
  to these tests.
- `scripts/check-no-php-in-server.sh` exits 1 if the server depends on any PHP crate. It runs in CI's `test` job. Not
  seen failing.

### Review checklist

- [ ] No feature keeps PHP state between requests.
- [ ] Every new way a worker can fail ends in kill, log and replace.
- [ ] Release switches still prepare the new pool before activating it.
- [ ] No PHP crate becomes a dependency of the server binary.

---

## Exceptions

None.

---

## Revision history

- **2026-11-18:** decided, in the PHP foundation design. The pools were a role of the server binary
  (`speedyspec-server php-worker`).
- **2026-11-19:** shipped in SpeedySpec/speedyspec-server#4.
- **2026-11-19:** superseded in part by the PHP runtime and admin gate design. The pools moved into their own binary,
  `speedyspec-php-runtime`, released and upgraded by the server like any app, so the server holds no PHP. The pool
  behaviour was kept. Shipped in SpeedySpec/speedyspec-server#6. As built, each pool also keeps two workers beyond
  `workers.max` for the server's own calls: one for health probes, one for gate calls and migrations.
- **2026-11-21:** recorded as an ADR, and rewritten in this structure.
