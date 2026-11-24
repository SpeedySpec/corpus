# ADR-014: WordPress is ported to Rust, and each ported area's PHP is deleted

**Status:** Accepted
**Date:** 2026-11-18
**Deciders:** the project owner
**Exploration:** [PHP foundation](../speedyspec-server/explorations/2026-11-18-php-foundation.md)
**Related:** [ADR-015](015-ports-and-adapters-with-cqrs-use-cases.md), [ADR-016](016-rust-runs-as-an-embedded-php-extension.md), [ADR-020](020-rust-owns-hook-state.md), [ADR-021](021-recorded-compatibility.md)

---

## Context

SpeedySpec runs a WordPress clone. Most of it is moving from PHP to Rust, and in the end only plugins will be PHP.
The move cannot happen in one step. It has to go one area at a time, such as hooks, then options, then the database.
Through every step, plugins and themes keep calling WordPress's functions by their usual names: `add_filter`,
`get_option`, `WP_Hook` and the rest.

### Why this matters

Without a rule, each area could end up with two implementations: WordPress's PHP and the Rust port. Two
implementations of one behaviour drift apart, and plugins would see different results depending on which one ran.
A PHP fallback would also keep PHP on the hot path, which is the cost the port exists to remove.

---

## Decision

WordPress's APIs are reimplemented in Rust and exposed to PHP as native functions and classes under WordPress's own
names. Once an area is ported and proven, WordPress's PHP for that area is deleted outright.

### Rules

1. Each WordPress API is reimplemented in Rust and registered with the PHP engine as native functions and classes.
2. WordPress's global functions stay, as thin shims over the Rust port.
3. A ported area's WordPress PHP is deleted once it is proven ([ADR-021](021-recorded-compatibility.md)). It is not
   kept as a package, a fallback, a vendored copy, or anywhere else.
4. Behaviour a plugin can observe does not change: callback order, globals, return values. The only allowed
   differences are the documented breaks listed in the area's design.
5. This is a port, not a compatible library. Laravel is not supported.

### What this looks like

✅ **Correct**

```rust
// crates/speedyspec-wp-hooks/src/messages.rs: add_filter() is a Rust command with a Rust handler.
pub struct AddFilter {
    pub hook: HookName,
    pub callback: Callback,
    pub priority: HookValue,
    pub accepted_args: i64,
}

impl CommandHandler<AddFilter> for HookApi {
    fn handle(&self, m: AddFilter) -> Result<bool, HookError> {
        self.add_filter(&m.hook, m.callback, &m.priority, m.accepted_args)
    }
}
```

❌ **Not permitted**

```php
// Keeping WordPress's PHP around "just in case".
if ( ! function_exists( 'add_filter' ) ) {
	require ABSPATH . WPINC . '/plugin.php';
}
```

### Patterns

- **Ports and Adapters** ([ADR-015](015-ports-and-adapters-with-cqrs-use-cases.md)): the WordPress function shims are
  driving adapters over the Rust ports.

---

## Consequences

### Positive

1. One implementation per behaviour. There is nothing to keep in step.
2. Hot paths such as `apply_filters` stay in Rust and need not cross into PHP.
3. Plugins and themes keep working, because the names and observable behaviour stay the same.

### Negative

1. Once an area's PHP is deleted, there is no fallback if a port has a bug.
   **Mitigation:** recorded compatibility and WordPress's own tests must pass before deletion, and stay as regression
   tests afterwards ([ADR-021](021-recorded-compatibility.md)).
2. Some behaviours cannot be reproduced exactly, such as replacing a hook global wholesale.
   **Mitigation:** each one is listed as a documented break in the area's design and in its `breaks.json`.
3. WordPress code cannot run without the Rust extension.
   **Mitigation:** none needed; this is intended ([ADR-016](016-rust-runs-as-an-embedded-php-extension.md)).

### Trade-offs

**We prioritise:**
- One implementation per behaviour.
- Rust speed on the hot paths.

**We accept:**
- No fallback to WordPress's PHP once an area is deleted.
- A short list of documented breaks.

---

## Fleet impact

- **speedyspec-server:** holds the Rust ports in its workspace, as crates with no server dependency.
- **speedyspec-php-runtime:** embeds the ports through the extension and serves them to PHP apps.
- **speedyspec-cli:** carries copies of the hook and container crates so WordPress plugins it runs keep calling
  `add_filter` ([ADR-023](023-shared-crates-in-the-server-workspace.md)).
- **wordpress-develop:** loses each area's PHP once that area is proven. For hooks, `functions/plugin.php`,
  `class-wp-hook.php` and the three PHP hook packages are gone.
- **WordPress plugins and themes:** must not depend on WordPress's PHP internals of a ported area beyond its public
  names. Unverified for third-party plugins.

---

## Enforcement

### Automated

- `cargo test -p speedyspec-wp-hooks-php --test php` replays the recorded hook scenarios against the port
  (`replayed_scenarios_match_wordpress`). Any unlisted mismatch makes the test binary exit 1, so `cargo test` fails.
- `cargo test -p speedyspec-wp-hooks-php --test wordpress` runs WordPress's own hook tests under a pinned PHPUnit in
  the embedded PHP. A failure fails `cargo test`.
- Both run in the server's CI, in the `php` job's "Test with php" step, which needs `scripts/build-php.sh` first.
  That step has failed in CI before; no failure has been traced to these tests.
- Nothing checks automatically that a ported area's PHP is gone from wordpress-develop. That part is review only.

### Review checklist

- [ ] The area's recorded scenarios and WordPress's own tests for it pass before any deletion.
- [ ] The deletion removes WordPress's implementation completely, with no copy kept elsewhere.
- [ ] Every behaviour difference is listed as a documented break in the design and in `breaks.json`.
- [ ] The WordPress function names stay callable from PHP.
- [ ] The project owner confirmed the deletion.

---

## Exceptions

Documented breaks are the only exceptions to rule 4. Each is listed in the area's design, and the project owner
approves the design. Deleting an area's PHP needs the project owner's confirmation.

---

## Revision history

- **2026-11-18:** decided, in the PHP foundation design.
- **2026-11-18:** the Hook API's PHP deleted from wordpress-develop, in the commit "refactor: remove WordPress's PHP
  Hook API; the Rust port replaces it".
- **2026-11-19:** the Hook API port shipped in SpeedySpec/speedyspec-server#4.
- **2026-11-21:** recorded as an ADR, and rewritten in this structure.
- **2026-11-23:** the documented breaks of rule 4 are now governed by [ADR-028](028-speedyspec-versions.md): in
  SpeedySpec 1 each one is also listed, with its workaround, in the departures register.
