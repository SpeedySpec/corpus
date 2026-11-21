# Exploration: Native plugin features

**Status:** built in part
**Created:** 2026-11-19
**Author:** Claude, for the project owner
**Context:** How to do the work of popular caching, security, SEO-plumbing and image plugins in Rust, without running PHP.
**Approved by:** the project owner on 2026-11-19
**Socialized with:** not socialized

---

## Problem Statement

Most WordPress sites run a few large plugins for page caching, firewall and login security, redirects and sitemaps,
and image optimization. Each one runs PHP on every request, often before WordPress itself loads. SpeedySpec's aim is
to keep as little PHP in flight per request as it can. A cache hit, a blocked request or a redirect should not start
the PHP interpreter at all. Nothing in SpeedySpec did this work yet, and the features had no agreed place to live.

## Current State

- In SpeedySpec/speedyspec-server, only WordPress's Hook API was ported to Rust (`speedyspec-wp-hooks`, from
  SpeedySpec/speedyspec-server#4). Database access was still PHP's `mysqli`.
- The PHP runtime ran each request straight into the PHP script. It had no step that could answer a request in Rust
  first.
- WordPress core already has a home for almost every feature: `redirect_canonical()` and old-slug redirects,
  `WP_Sitemaps`, `do_robots()` and `wp_robots()`, the feed templates, the `clean_*_cache()` calls on every change,
  the `authenticate` filter and `wp_login_failed`, and `WP_Image_Editor` with resumable sub-sizes.
- The plugins hook in early: Wordfence through `auto_prepend_file`, page caches through `advanced-cache.php` when
  `WP_CACHE` is set.

## Proposed Solution

Each feature becomes behaviour of the WordPress API it extends, built in Rust once that API is ported. The runtime
answers requests in fixed stages before any PHP runs.

### Architecture

```
FastCGI request (server -> PHP runtime worker)
        |
        v
 1. Request rules (firewall) ----- block --> 403 / 429
        |
 2. Response cache lookup -------- hit ----> cached page
        |
 3. Routing ---------------------- redirect -> 301/302/307/410
        |                           not found -> 404 log
        v
 4. Native responders (sitemaps, robots.txt, feeds)
        |  (only if no PHP callback is registered on the hooks they fire)
        v
    PHP script
```

- Request rules run for every PHP app (blog, front-end API, admin API). The cache and native responders run for the
  blog and front-end API only.
- Each worker loads its app's plugins and theme once at start and records which hooks have PHP callbacks. A native
  path that would fire one of those hooks hands the request to PHP instead.
- Stores (page cache, firewall counters and bans, redirect rules, 404 log, job queue) are contracts with file, Redis
  and database implementations, plus an in-memory one for tests.
- A background job runner lives inside the runtime beside the request workers. It runs image work and scheduled
  events, and is meant to replace request-driven cron.

### Data Model

The firewall, the one feature built so far, keeps its settings as 13 `speedyspec_firewall_*` options read through the
ported options API. The firewall is off by default. Counters and bans are entries in any Caching Store, keyed by
client and fixed time window, and expire on their own.

### Interfaces

The generic stage contract from `speedyspec-native-stage`, which the page cache is meant to reuse:

```rust
pub enum StageOutcome { Continue, Respond(NativeResponse) }
pub trait NativeStage {
    fn name(&self) -> &str;
    fn answer(&self, request: &StageRequest<'_>) -> StageOutcome;
}
```

A stage sees only the FastCGI params. `StageChain` asks each stage in turn; the first `Respond` wins.

## Design Decisions

1. **Features are behaviour of the ported APIs, not separate Rust plugins.** Core already has a home for almost every
   feature and already calls its cache-clearing functions on every change, so invalidation falls out of the ports.
   See [ADR-001](../../adr/001-plugin-features-are-behaviour-of-ported-apis.md).
   *Trade-off:* a feature can't ship until the API it belongs to is ported.
2. **Everything runs in the runtime's Rust side, before PHP.** Chosen by the project owner to see how it runs. See
   [ADR-002](../../adr/002-native-features-run-in-the-runtime.md).
   *Trade-off:* blocked and cached requests still cross FastCGI into the runtime.
3. **Wait for real Rust ports for data.** Chosen by the project owner. See
   [ADR-003](../../adr/003-native-features-wait-for-api-ports.md).
   *Trade-off:* porting the database, options, permalinks, posts and more comes first.
4. **Fixed request stages: rules, cache, routing, responders.** See
   [ADR-004](../../adr/004-runtime-request-stages.md).
   *Trade-off:* static admin-site files never reach the runtime, so the firewall doesn't cover them.
5. **Core hook names and order stay; new hooks use the project's prefix.** See
   [ADR-005](../../adr/005-core-hooks-stay.md).
   *Trade-off:* the original plugins' own PHP APIs and option names are not supported.
6. **A PHP callback registry built at worker start.** Chosen by the project owner. See
   [ADR-006](../../adr/006-php-callback-registry-at-worker-start.md).
   *Trade-off:* a callback registered only on some requests still sends every such request to PHP.
7. **Shared stores with file, Redis and database implementations.** Chosen by the project owner; the designer set
   the defaults. See [ADR-007](../../adr/007-shared-stores-file-redis-database.md).
   *Trade-off:* three implementations to keep for each store.
8. **A long-lived background job runner in the runtime.** Chosen by the project owner. See
   [ADR-008](../../adr/008-background-job-runner-in-the-runtime.md).
   *Trade-off:* one more supervised process per runtime.
9. **The database port must support several engines, a query builder that parses raw SQL, a unit of work and plain
   objects.** Stated by the project owner. See [ADR-009](../../adr/009-database-port-requirements.md).
   *Trade-off:* ReDB has no SQL, so it needs its own repository implementations.
10. **Firewall and login security as request rules plus login protection.** See
    [ADR-010](../../adr/010-firewall-and-login-security.md).
    *Trade-off:* until an auth port exists, lockouts are enforced at the request stage, not in `authenticate`.
11. **The page cache purges everything first, by tag later.** The designer's call. See
    [ADR-011](../../adr/011-page-cache-coarse-purge-first.md).
    *Trade-off:* every content change empties the whole cache until the post and term ports land.
12. **Redirects, sitemaps, robots and feeds extend core's own subsystems.** See
    [ADR-012](../../adr/012-redirects-sitemaps-robots-feeds.md).
    *Trade-off:* feed hooks that print need output buffering when a PHP callback is present.
13. **Images in Rust, with sub-sizes on the job runner.** See
    [ADR-013](../../adr/013-images-in-rust-on-the-job-runner.md).
    *Trade-off:* sub-sizes appear after the upload returns, not with it.

Decisions the firewall build added:

14. **The admin API gets its own firewall scope with no rate limit and no IP bans.** The block editor sends bursts of
    requests, and a ban could lock the site's owner out of their own admin. Lists, patterns and login lockout still
    apply there.
    *Trade-off:* the admin is not rate-limited at all.
15. **IPv6 clients are counted and banned per /64.** A client rotating addresses inside one network would otherwise
    escape every counter.
    *Trade-off:* neighbours in one /64 share a limit.
16. **Windows, bans and lockouts are capped at 30 days.** A typo can't ban a client for good.
    *Trade-off:* no permanent bans through these settings; the block list covers that.
17. **The firewall fails open.** A store error, unreadable settings or a missing client address lets the request
    through and logs a warning.
    *Trade-off:* an outage of the store turns the firewall off.
18. **User-enumeration blocking counts only credentials WordPress checks itself.** Review of the branch found
    bypasses: a forged `Authorization` header or a fake logged-in cookie exempted a request, and
    `?rest_route=/wp/v2/users` and percent-encoded `author` values slipped past. Now only HTTP Basic (application
    passwords) or a logged-in cookie with a REST nonce exempts a request, those forms are caught, and admin screens
    such as `edit.php?author=1` are left alone.
    *Trade-off:* a logged-in browser without a REST nonce is treated as anonymous.
19. **Login lockouts are enforced at the request stage for now.** No auth port exists, so `authenticate` can't
    enforce them. A `POST` to a configured sign-in path from a locked-out client gets 429.
    *Trade-off:* the check is by path, not by the sign-in code itself.

## Implementation Phases

1. **Database layer and options ports.** Built elsewhere, in SpeedySpec/speedyspec-server#12 to #19.
   *Delivers:* what the firewall needs. Shipped.
2. **The firewall.** Six crates: `speedyspec-native-stage`, `speedyspec-firewall`, `-app`, `-infra-cache`,
   `-infra-options` and `-presentation`, plus a wiring request for the runtime's owner.
   *Delivers:* a firewall that can answer before PHP. Merged as
   [SpeedySpec/speedyspec-server#20](https://github.com/SpeedySpec/speedyspec-server/pull/20) on 2026-11-21. Not yet
   wired into the runtime's worker.
3. **Permalinks and rewrite.** *Delivers:* the page cache (coarse purge) and redirects. Not started.
4. **Attachments and the background job runner.** *Delivers:* image optimization. Not started.
5. **Posts, a `WP_Query` subset, terms, users.** *Delivers:* sitemaps, feeds, `robots.txt` contributions, and
   tag-based cache purging. Not started.

## Fleet Impact

- **PHP runtime:** gains the stage chain and every native feature. Confirmed for the firewall's crates in
  SpeedySpec/speedyspec-server#20; the runtime's worker itself was not changed there.
- **Blog and front-end API:** get all four stages. Unverified until the runtime is wired.
- **Admin API:** gets request rules in the admin scope, and login protection on its sign-in endpoint once it exists.
  Unverified.
- **Server:** unchanged by design. It holds no PHP or WordPress knowledge.
- **Admin site, CLI:** unchanged. Admin-site static files are served by the server and never reach the firewall.

## Open Questions

1. **When does the runtime wire in the stage chain?** Settled when the runtime's owner acts on the firewall's wiring
   request.
2. **How are body patterns checked?** A stage must not read the body. Needs a design that can inspect it without
   taking it from PHP.
3. **How do PHP plugins add request rules?** Deferred until the worker-start callback registry exists.
4. **When does `authenticate` enforce lockouts?** Deferred until an auth port and the admin API's sign-in endpoint
   exist; the sign-in path is then added to `speedyspec_firewall_signin_paths`.
5. **Should the firewall cover static admin-site files?** Revisit if the split server-and-runtime approach is tried.
6. **How does only one replica run a given job under `[cluster]`?** Left to the job runner's own design.
7. **Is the rate limit's undercount acceptable long term?** On Valkey and ReDB the increment reads then writes, so
   it can undercount by one under concurrency. Settled by an atomic increment in the cache contract, if needed.

## Success Criteria

- [x] The firewall's rules block listed IPs with 403 and rate-limited clients with 429 and `Retry-After`. Checked by
  the 47 tests in the six crates (SpeedySpec/speedyspec-server#20).
- [x] An allow-listed client is never blocked or counted. Checked by `allow_list_wins_over_everything`.
- [x] A failing store lets every request through. Checked by `store_errors_fail_open`.
- [x] IPv4-mapped IPv6 clients match IPv4 entries. Checked by `ip_set_matches_ipv4_mapped_ipv6`.
- [x] Counters and bans work on the memory, key-value and file caches, and live on Valkey. Checked by the adapter
  crate's backend tests.
- [x] The firewall's domain crate depends only on `ipnet` and `regex`. Checked with `cargo tree`.
- [ ] A blocked, cached or redirected request never starts the PHP script. Needs the runtime wiring.
- [ ] Recorded compatibility tests for canonical redirects, sitemaps, `robots.txt`, feeds and image sub-sizes. Needs
  those ports.

---

## Non-Goals

- **Compatibility with the original plugins' PHP APIs and option names** — SpeedySpec uses its own; importing their
  settings is optional.
- **CDN integration, CSS and JS minification, critical CSS, database cleanup, malware scanning, two-factor auth, SEO
  scoring** — outside the four plugin families.
- **A cache or firewall in the server, in front of FastCGI** — a later experiment.
- **Designing the API ports themselves** — each port gets its own design and is owned by other sessions.

## Alternatives Considered

### Option: Split between server and runtime
- **Pros:** Fastest path; blocked and cached requests never reach the runtime.
- **Cons:** The server would act on what each app tells it, which blurs its boundary.
- **Verdict:** Kept as a later experiment.

### Option: Everything in the server
- **Pros:** Simplest to reason about.
- **Cons:** Gives the server database access and WordPress knowledge, overturning the runtime design.
- **Verdict:** Rejected.

### Option: A Rust read layer over WordPress's tables
- **Pros:** Fast, and available at once.
- **Cons:** Ties Rust to the table layout and misses anything plugins filter in PHP.
- **Verdict:** Rejected.

### Option: Call into PHP for data
- **Pros:** No porting needed.
- **Cons:** Runs PHP again, which defeats the purpose.
- **Verdict:** Rejected.

### Option: Separate Rust plugin packages
- **Pros:** Features stay apart from the ports.
- **Cons:** Reimplements cache invalidation and image generation beside the APIs that already trigger them.
- **Verdict:** Rejected in favour of behaviour of the ported APIs.

## Dependencies

- The database layer and options ports, for the firewall.
- The permalinks, attachments, posts, query, terms and users ports, and the job runner, for the other features.
- The Caching Store and its backends, which hold the firewall's counters and bans.
- The worker-start PHP callback registry, for any native path that fires a hook PHP may also use.

## References

- [ADR-001](../../adr/001-plugin-features-are-behaviour-of-ported-apis.md),
  [0002](../../adr/002-native-features-run-in-the-runtime.md),
  [0003](../../adr/003-native-features-wait-for-api-ports.md),
  [0004](../../adr/004-runtime-request-stages.md),
  [0005](../../adr/005-core-hooks-stay.md),
  [0006](../../adr/006-php-callback-registry-at-worker-start.md),
  [0007](../../adr/007-shared-stores-file-redis-database.md),
  [0008](../../adr/008-background-job-runner-in-the-runtime.md),
  [0009](../../adr/009-database-port-requirements.md),
  [0010](../../adr/010-firewall-and-login-security.md),
  [0011](../../adr/011-page-cache-coarse-purge-first.md),
  [0012](../../adr/012-redirects-sitemaps-robots-feeds.md),
  [0013](../../adr/013-images-in-rust-on-the-job-runner.md)
- [SpeedySpec/speedyspec-server#20](https://github.com/SpeedySpec/speedyspec-server/pull/20), the firewall
- Source spec: `2026-11-19-native-plugin-features-design.md`
- Source plan: `2026-11-20-native-firewall.md`
