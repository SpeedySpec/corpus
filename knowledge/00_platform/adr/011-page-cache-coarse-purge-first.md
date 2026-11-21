# ADR-011: The page cache purges everything first, by tag later

**Status:** Accepted
**Date:** 2026-11-19
**Deciders:** the designer
**Exploration:** [Native plugin features](../speedyspec-server/explorations/2026-11-19-native-plugin-features.md)
**Related:** [ADR-001](001-plugin-features-are-behaviour-of-ported-apis.md), [ADR-003](003-native-features-wait-for-api-ports.md), [ADR-004](004-runtime-request-stages.md), [ADR-005](005-core-hooks-stay.md), [ADR-007](007-shared-stores-file-redis-database.md)

---

## Context

Core has no page cache of its own, only the `advanced-cache.php` drop-in point. But core already calls
`clean_post_cache()`, `clean_term_cache()`, `clean_user_cache()` and `wp_cache_flush()` on every change. Those calls
are the natural place to purge a page cache.

The timing is awkward. The page cache can ship at step 2 of ADR-003, with permalinks. The ports that call the
`clean_*_cache` functions (posts, terms, users) arrive only at step 4.

### Why this matters

A page cache that misses a change serves stale content, which is worse than no cache. Purging precisely needs to know
what each page depends on, and that knowledge comes from the step 4 ports. Waiting for step 4 would delay the cache;
shipping precise purging early would mean guessing.

---

## Decision

The page cache ships at step 2 and purges everything on any content change. From step 4, each cached response is
tagged with what it depends on, and changes purge by tag.

### Rules

1. Until step 4, any content change empties the whole cache.
2. From step 4, each cached response carries tags: post IDs, term IDs, author, home page, feeds, sitemaps.
3. From step 4, the ported `clean_*_cache` functions, `transition_post_status`, `switch_theme` and `updated_option`
   purge by tag.
4. Purges are Rust hooks in the shared Hook API. They run in whichever pool made the change, so the cache store must
   be reachable by every pool that serves or changes content (ADR-007).
5. Requests bypass the cache when they carry logged-in or comment-author cookies, have a configured query string, are
   not `GET`, or set `DONOTCACHEPAGE`.
6. Kept or added: a cache-key filter, and a purge-by-URL-or-tag command for plugins that change content behind
   core's back.
7. The admin API is never cached (ADR-004).

### What this looks like

✅ **Correct**

```rust
// Step 2: a Rust callback on core's own change hook empties the cache.
// (Actions are registered with add_filter, as in WordPress.)
hooks.add_filter(&HookName::from("clean_post_cache"), Callback::rust(
    "speedyspec_page_cache_purge",
    move |_| { page_cache.purge_all()?; Ok(HookValue::Null) },
), &HookValue::Int(10), 1)?;
```

❌ **Not permitted**

```rust
// Invalidation kept apart from core: a timer that guesses, or a purge a write path can skip.
every(Duration::from_secs(300), || page_cache.purge_all());
```

### Patterns

- **Observer:** purging, through the Hook API.
- **Strategy:** the store backends.

---

## Consequences

### Positive

1. The cache can ship as soon as permalinks are ported.
2. Purging is never less than needed, so stale pages are not served.
3. Purging rides on core's own change calls, so no write path is missed.

### Negative

1. Until step 4, every edit empties the whole cache, and the next visitors pay for rebuilding it.
   **Mitigation:** tag-based purging at step 4.
2. Plugins that change content without calling core's functions leave stale pages.
   **Mitigation:** the purge-by-URL-or-tag command.

### Trade-offs

**We prioritise:**
- correct pages over hit rate;
- shipping early.

**We accept:**
- low hit rates on sites that are edited often, until step 4.

---

## Fleet impact

- **speedyspec-server (repository):** gains the page cache with the permalinks port, and tags with the step 4 ports.
  Not started.
- **speedyspec-php-runtime:** runs the cache as stage 2 for the blog and the front-end API; every pool must reach the
  same cache store. Not built.
- **Blog, front-end API:** served from the cache.
- **Admin API:** never cached, but its pool runs most purges.
- **Plugins:** may use the cache-key filter and the purge command.
- **speedyspec-server (binary), speedyspec-cli, admin site:** unaffected.

---

## Enforcement

### Automated

None. This rule is enforced by review only.

### Review checklist

- [ ] Every content change reaches a purge through core's hooks.
- [ ] Before step 4, purges empty the whole cache; nothing purges partially without tags.
- [ ] The bypass rules are applied before a cache lookup.
- [ ] The cache store is configured so every pool of the app reaches it.

---

## Exceptions

None.

---

## Revision history

- **2026-11-19:** decided, in the native plugin features design.
- **2026-11-21:** recorded as an ADR, and rewritten in this structure.
