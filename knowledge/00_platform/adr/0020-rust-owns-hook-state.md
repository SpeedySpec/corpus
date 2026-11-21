# 0020. Rust owns hook state; WordPress's hook globals are proxies

- **Status:** Accepted, 2026-11-18
- **Decided by:** the project owner

## Context

The Hook API was the first area ported (0014). Plugins read and change `$wp_filter`, `$wp_current_filter`,
`$wp_actions` and `$wp_filters` directly, and Rust callers must never read PHP.

## Decision

- The hook store is Rust. The four globals are native proxy objects over it, so reading, indexing, iterating and
  appending work as in WordPress.
- `WP_Hook` stays as a native class with `ArrayAccess`, `Iterator` and `Countable`; its public methods map to the
  same commands and queries.
- Callbacks are Rust (worker lifetime) or PHP (request lifetime), held in one store in one priority order. Calling a
  Rust callback never enters PHP.
- The `all` hook, `accepted_args` and priorities that aren't integers behave as WordPress does.

## Consequences

- **Documented break:** replacing one of those globals wholesale (`$wp_current_filter = array()`, or a test harness
  swapping `$GLOBALS['wp_filter']`) replaces the proxy, and the Rust store doesn't see it.
- WordPress's test harness saves and restores the Rust store instead. Hooks set before the API loads
  (`build_preinitialized_hooks`) are imported when a request starts.

## Patterns

- **Proxy:** the globals.
- **Repository:** `HookStore`, with read and write sides.
- **Observer:** the hooks themselves.
