# ADR-006: A registry of PHP callbacks, built when a worker starts

**Status:** Accepted
**Date:** 2026-11-19
**Deciders:** the project owner
**Exploration:** [Native plugin features](../speedyspec-server/explorations/2026-11-19-native-plugin-features.md)
**Related:** [ADR-004](004-runtime-request-stages.md), [ADR-005](005-core-hooks-stay.md), [ADR-010](010-firewall-and-login-security.md), [ADR-012](012-redirects-sitemaps-robots-feeds.md)

---

## Context

In WordPress, PHP callbacks exist for one request only. PHP registers them as it loads plugins and the theme. A
request answered in Rust never loads plugins or the theme, so it never sees their callbacks. Yet core hooks must keep
working (ADR-005): if a theme filters `robots_txt`, a `robots.txt` served from Rust must not skip that filter.

### Why this matters

Without a way to know which hooks PHP cares about, a native path has two bad choices. It can ignore PHP callbacks and
serve wrong output. Or it can always fall back to PHP, and then the native path saves nothing.

---

## Decision

Each runtime worker loads its app's plugins and theme once, at startup, and records which hooks have PHP callbacks.
Native paths consult that registry before staying in Rust.

### Rules

1. A native path (redirect, sitemap, `robots.txt`, feed, cache-key check, firewall rule) checks the registry for
   every hook it would fire.
2. If none of those hooks has a PHP callback, the request stays in Rust. If any does, the request goes to PHP, and
   the PHP path fires the hook as usual.
3. A callback registered only on some requests is still recorded. The registry may send a request to PHP more often
   than needed, never less.
4. The registry is rebuilt whenever a worker is replaced. Plugin activation, deactivation and new releases all
   recycle workers.
5. Rust callbacks a feature registers go into the hook store's worker partition, at worker start, so they outlive
   each request.
6. Rules added by PHP plugins (for example firewall request rules) wait for this registry.

### What this looks like

✅ **Correct**

```rust
// A native responder asks the registry before answering in Rust.
fn answer(&self, request: &StageRequest<'_>) -> StageOutcome {
    if !request_is_robots_txt(request) {
        return StageOutcome::Continue;
    }
    if self.php_callbacks.any(&["robots_txt", "do_robotstxt"]) {
        return StageOutcome::Continue; // PHP serves it and fires the hooks
    }
    StageOutcome::Respond(self.robots.build())
}
```

❌ **Not permitted**

```rust
// Serving from Rust without checking: a theme's robots_txt filter is skipped.
fn answer(&self, request: &StageRequest<'_>) -> StageOutcome {
    StageOutcome::Respond(self.robots.build())
}
```

### Patterns

- **Registry:** built once per worker, read on every native path.

---

## Consequences

### Positive

1. Native paths stay correct in the presence of PHP plugins and themes.
2. Most requests on a site with few PHP callbacks stay in Rust.

### Negative

1. A conditionally registered callback sends every matching request to PHP, even when the condition would be false.
   **Mitigation:** none. Correctness wins over speed.
2. Worker start is slower, because it loads all plugins and the theme.
   **Mitigation:** none yet.
3. Native features that would fire hooks with PHP callbacks can't be fully native until the registry exists.
   **Mitigation:** the firewall defers PHP-added request rules until then.

### Trade-offs

**We prioritise:**
- correct output over the fastest path;
- a registry that is never stale, by tying it to the worker's life.

**We accept:**
- extra trips to PHP for conditional callbacks;
- slower worker start.

---

## Fleet impact

- **speedyspec-php-runtime:** must load plugins and the theme at worker start, build the registry and expose it to
  native paths. Not built yet.
- **speedyspec-server (repository):** the hook store already has a worker partition (`Partition::Worker` in
  `speedyspec-wp-hooks-memory`); feature callbacks must be registered there. The registry itself is unverified.
- **Blog, front-end API:** native responders and the cache key check depend on it.
- **Admin API:** request rules depend on it once PHP plugins can add them.
- **Plugins:** keep working; their callbacks route requests to PHP.
- **speedyspec-server (binary), speedyspec-cli, admin site:** unaffected.

---

## Enforcement

### Automated

None. This rule is enforced by review only.

### Review checklist

- [ ] Every native path lists the hooks it fires and checks each against the registry.
- [ ] A hit in the registry sends the request to PHP; nothing approximates the PHP callback in Rust.
- [ ] Feature callbacks are registered at worker start, in the worker partition.
- [ ] Nothing caches the registry beyond the worker's life.

---

## Exceptions

None.

---

## Revision history

- **2026-11-19:** decided, in the native plugin features design.
- **2026-11-21:** recorded as an ADR, and rewritten in this structure.
