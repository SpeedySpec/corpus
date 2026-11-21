# ADR-016: Rust runs as a PHP extension compiled into the embedded PHP

**Status:** Accepted
**Date:** 2026-11-18
**Deciders:** the designer, within the project owner's rule that no PHP sits between callers and Rust
**Exploration:** [PHP foundation](../speedyspec-server/explorations/2026-11-18-php-foundation.md)
**Related:** [ADR-014](014-port-wordpress-to-rust-and-delete-its-php.md), [ADR-017](017-php-twins-and-the-override-flag.md), [ADR-024](024-static-php-with-every-wordpress-extension.md), [ADR-025](025-supervised-php-worker-pools.md)

---

## Context

The Rust ports ([ADR-014](014-port-wordpress-to-rust-and-delete-its-php.md)) must be callable from the PHP that
remains: WordPress's leftover code and plugins. Rust callers must not pay for PHP at all. The CLI already embedded a
static PHP with a Rust extension built on `ext-php-rs`, so the approach was known to work.

### Why this matters

If PHP code sat between a caller and Rust, every `apply_filters` would cross into PHP and back. That is the cost the
port is meant to remove. If Rust callers had to convert values to PHP, Rust code would be tied to a PHP engine even
when no PHP is involved.

---

## Decision

The Rust code is a PHP extension, written with `ext-php-rs` and compiled into the embedded, static PHP. Native
functions and classes run Rust directly, and Rust calls Rust without going through PHP.

### Rules

1. The ports are exposed to PHP as native functions and classes of one extension, built with `ext-php-rs`.
2. The extension is compiled into the static, embedded PHP ([ADR-024](024-static-php-with-every-wordpress-extension.md)).
3. Rust code calls Rust directly. It never reaches a Rust adapter through PHP.
4. Rust code never converts values to PHP when no PHP adapter is plugged in.
5. The extension is named `speedyspec_wordpress`, so it never collides with the CLI's `speedyspec` module.

### What this looks like

✅ **Correct**

```rust
// crates/speedyspec-php/src/hook_class.rs
#[php_module]
#[php(startup = startup)]
pub fn module(module: ModuleBuilder) -> ModuleBuilder {
    let mut module = module
        .name("speedyspec_wordpress")
        .request_shutdown_function(crate::area::request_shutdown);
    for function in crate::area::functions() {
        module = module.function(function);
    }
    module
}
```

❌ **Not permitted**

```php
// A PHP library in front of Rust, through FFI.
$ffi = FFI::cdef( 'int add_filter(const char *hook);', 'libspeedyspec.so' );
function add_filter( $hook, $callback ) { return $GLOBALS['ffi']->add_filter( $hook ); }
```

### Patterns

- **Adapter:** the native functions and classes are driving adapters over the ports.

---

## Consequences

### Positive

1. A PHP call to `add_filter` runs Rust with no PHP in between.
2. A Rust caller with only Rust callbacks never enters PHP.
3. The host needs no PHP installed: everything is in one static binary.

### Negative

1. WordPress code cannot run under an external php-fpm: without the extension it has no WordPress API.
   **Mitigation:** none; it is a non-goal.
2. The Rust is tied to the embedded build: no Windows, no thread-safe PHP, no threads running PHP.
   **Mitigation:** none; these are out of scope.
3. Building needs the static PHP first, which is slow.
   **Mitigation:** CI caches the static PHP build, keyed on `php-version` and `scripts/build-php.sh`.

### Trade-offs

**We prioritise:**
- No PHP between callers and Rust.
- A host with no PHP installed.

**We accept:**
- A custom PHP build.
- No external php-fpm, no Windows, no ZTS.

---

## Fleet impact

- **speedyspec-php-runtime:** embeds the static PHP with `speedyspec_wordpress` and serves the PHP apps. This is the
  only binary that runs WordPress PHP on the server side.
- **speedyspec-server:** holds no PHP since the PHP runtime design ([ADR-025](025-supervised-php-worker-pools.md)).
- **speedyspec-cli:** has its own extension, `speedyspec`, and carries the copied crates
  ([ADR-023](023-shared-crates-in-the-server-workspace.md)). The two names must not collide.
- **PHP apps, plugins and themes:** must run inside the runtime. They cannot be served by a stock php-fpm.

---

## Enforcement

### Automated

- `cargo test -p speedyspec-wp-hooks-php --test php` runs `rust_caller_with_rust_callbacks_never_enters_php`: it
  fires a hook from Rust and checks the PHP boundary counter did not move. A crossing makes the binary exit 1.
- `cargo test -p speedyspec-php-ext --test ext` runs `module_name_is_speedyspec_wordpress`. A different name exits 1.
- Both run in the server's CI `php` job ("Test with php"). Not confirmed seen failing.

### Review checklist

- [ ] New WordPress names are native functions or classes, not PHP files calling Rust.
- [ ] Rust callers reach Rust adapters directly, with no PHP round trip.
- [ ] No value is converted to PHP unless a PHP adapter is bound.
- [ ] Any new hot path has a boundary-counter test.

---

## Exceptions

A PHP adapter bound to a port ([ADR-017](017-php-twins-and-the-override-flag.md)) is the only case where a Rust call
enters PHP. No other exception.

---

## Revision history

- **2026-11-18:** decided, in the PHP foundation design. FFI was rejected because it puts PHP between the caller and
  Rust.
- **2026-11-19:** shipped in SpeedySpec/speedyspec-server#4. SpeedySpec/speedyspec-server#5 then removed the last
  PHP crossing from an empty `apply_filters`.
- **2026-11-21:** recorded as an ADR, and rewritten in this structure.
