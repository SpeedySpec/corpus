# 0011. The page cache purges everything first, by tag later

- **Status:** Accepted, 2026-11-19
- **Decided by:** the designer

## Context

Core has no page cache beyond the `advanced-cache.php` drop-in. But it calls `clean_post_cache()`,
`clean_term_cache()`, `clean_user_cache()` and `wp_cache_flush()` on every change. The cache arrives with step 2 of
0003, while the ports that call the `clean_*_cache` functions arrive at step 4.

## Decision

- The cache ships at step 2 and purges everything on any content change.
- From step 4, each cached response is tagged with what it depends on: post IDs, term IDs, author, home page, feeds,
  sitemaps. The ported `clean_*_cache` functions, `transition_post_status`, `switch_theme` and `updated_option` then
  purge by tag.
- Requests bypass the cache when they carry logged-in or comment-author cookies, have a configured query string, are
  not `GET`, or set `DONOTCACHEPAGE`.
- Kept or added: a cache-key filter, and a purge-by-URL-or-tag command for plugins that change content behind core's
  back.

## Consequences

- Purges are Rust hooks in the shared Hook API, so they happen in whichever pool made the change, usually the admin
  API's. The cache store must be reachable by every pool that serves or changes content (0007).
- Until step 4, every edit empties the whole cache.

## Patterns

- **Observer:** purging, through the Hook API.
- **Strategy:** the store backends.
