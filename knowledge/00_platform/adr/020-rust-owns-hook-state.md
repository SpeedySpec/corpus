# ADR-020: Rust owns hook state; WordPress's hook globals are proxies

**Status:** Accepted
**Date:** 2026-11-18
**Deciders:** the project owner
**Exploration:** [PHP foundation](../speedyspec-server/explorations/2026-11-18-php-foundation.md)
**Related:** [ADR-014](014-port-wordpress-to-rust-and-delete-its-php.md), [ADR-017](017-php-twins-and-the-override-flag.md), [ADR-019](019-command-and-query-buses.md), [ADR-021](021-recorded-compatibility.md)

---

## Context

The Hook API was the first area ported ([ADR-014](014-port-wordpress-to-rust-and-delete-its-php.md)). In WordPress,
hook state lives in four globals: `$wp_filter`, `$wp_current_filter`, `$wp_actions` and `$wp_filters`. Plugins read
and change them directly, and WordPress's own test harness swaps them. Rust callers, on the other hand, must never
read PHP to fire a hook.

### Why this matters

If PHP owned the state, every Rust `apply_filters` would read PHP arrays, and the port would gain nothing. If Rust
owned it with no way in from PHP, plugins that read `$wp_filter` would break. Either choice decides which behaviours
can stay exact and which become documented breaks.

---

## Decision

The hook store is Rust. WordPress's hook globals are native proxy objects over it, and `WP_Hook` stays as a native
class, so PHP code that reads or iterates them keeps working.

### Rules

1. Hook state lives in one Rust store (`HookStore`).
2. `$wp_filter`, `$wp_actions` and `$wp_filters` are native proxies: reading, indexing, `isset`, iterating, counting
   and appending work as in WordPress.
3. `WP_Hook` stays as a native class with `ArrayAccess`, `Iterator` and `Countable`. Its public methods map to the
   same commands and queries as the functions.
4. Callbacks are Rust (worker lifetime) or PHP (request lifetime), held in one store in one priority order.
5. Calling a Rust callback never enters PHP.
6. The `all` hook, `accepted_args` and priorities that are not integers behave as in WordPress.
7. Replacing a hook global wholesale is a documented break: the Rust store does not see it.
8. Hooks set before the API loads (`build_preinitialized_hooks`) are imported when a request starts.

### What this looks like

✅ **Correct**

```php
// Reading and iterating the proxies works as in WordPress.
global $wp_filter;
if ( isset( $wp_filter['init'] ) ) {
	foreach ( $wp_filter['init'] as $priority => $callbacks ) {
		// ...
	}
}
```

❌ **Not permitted**

```php
// Replacing a global wholesale: the Rust store never sees it.
$GLOBALS['wp_filter'] = array();
// Passing a proxy to an array function throws a TypeError.
$names = array_keys( $wp_filter );
```

### Patterns

- **Proxy:** the hook globals.
- **Repository:** `HookStore`, with read and write sides.
- **Observer:** the hooks themselves.

---

## Consequences

### Positive

1. Rust code fires hooks without reading PHP.
2. Plugins that read, index and iterate `$wp_filter` keep working.
3. Rust and PHP callbacks share one priority order.

### Negative

1. Replacing one of the globals wholesale replaces the proxy, and the Rust store does not see it.
   **Mitigation:** listed in `compat/hooks/breaks.json`; WordPress's test harness saves and restores the Rust store
   instead.
2. Array functions such as `array_keys` throw a `TypeError` on the proxies.
   **Mitigation:** recorded as a known limit in `breaks.json`; real plugin use will show whether it matters.
3. Assigning anything but a `WP_Hook` to `$wp_filter[$name]` throws a `TypeError`.
   **Mitigation:** listed as a documented break.

### Trade-offs

**We prioritise:**
- Rust callers that never read PHP.
- Plugins that read the globals.

**We accept:**
- Wholesale replacement and array functions on the globals do not work as in WordPress.

---

## Fleet impact

- **speedyspec-php-runtime:** registers the proxies and the native `WP_Hook` in every request.
- **speedyspec-cli:** carries the same hook crates, so plugins it runs see the same proxies
  ([ADR-023](023-shared-crates-in-the-server-workspace.md)).
- **WordPress plugins and themes:** must not replace the hook globals or pass them to array functions. Whether any
  plugin in use does so is unverified.
- **wordpress-develop's test harness:** saves and restores the Rust store, not the globals.

---

## Enforcement

### Automated

- `cargo test -p speedyspec-wp-hooks-php --test php` replays the hook scenarios. Steps listed in
  `compat/hooks/breaks.json` are allowed to differ; any other difference exits 1. The same binary checks
  `rust_caller_with_rust_callbacks_never_enters_php` and `php_and_rust_callbacks_share_one_priority_order`.
- `cargo test -p speedyspec-wp-hooks-php --test wordpress` runs WordPress's own hook tests. Tests marked
  `@group documented-break` are excluded and must each be listed in `breaks.json`.
- Both run in the server's CI `php` job. Not confirmed seen failing.

### Review checklist

- [ ] New hook behaviour keeps state in the Rust store, not in PHP.
- [ ] Any new difference from WordPress is listed in `breaks.json` and the area's design.
- [ ] Rust callbacks stay worker lifetime and PHP callbacks request lifetime.

---

## Exceptions

The documented breaks in `compat/hooks/breaks.json` are the exceptions: replacing `$wp_filter`, `$wp_filters` or the
result of `build_preinitialized_hooks` wholesale, and assigning a non-`WP_Hook` to `$wp_filter[$name]`. The project
owner approved them in the design.

---

## Revision history

- **2026-11-18:** decided, in the PHP foundation design.
- **2026-11-19:** shipped in SpeedySpec/speedyspec-server#4. As built, `$wp_current_filter` stayed a real PHP array,
  updated in place to match the store; the other three globals are proxies.
- **2026-11-19:** SpeedySpec/speedyspec-server#5 made an empty `apply_filters` cross into PHP zero times.
- **2026-11-21:** recorded as an ADR, and rewritten in this structure.
