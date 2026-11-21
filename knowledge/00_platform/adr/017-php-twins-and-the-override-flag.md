# ADR-017: Every port has a PHP twin, and binding it to PHP is the override

**Status:** Accepted
**Date:** 2026-11-18
**Deciders:** both: the project owner (PHP may override a port behind a flag); the designer (the rule for stateful ports)
**Exploration:** [PHP foundation](../speedyspec-server/explorations/2026-11-18-php-foundation.md)
**Related:** [ADR-015](015-ports-and-adapters-with-cqrs-use-cases.md), [ADR-016](016-rust-runs-as-an-embedded-php-extension.md), [ADR-018](018-a-service-container-in-rust.md), [ADR-020](020-rust-owns-hook-state.md)

---

## Context

The project owner wanted PHP to keep the power to implement a contract and override the Rust one, behind a flag.
With the flag unset, a call must stay in Rust only. A port is a Rust trait, so PHP needs a matching interface to
implement, and the system needs to know, per port, whether PHP or Rust is answering.

### Why this matters

A hand-written PHP interface would drift from its Rust trait, and a PHP override would break at run time. Without a
cheap flag, every call would have to check PHP. A store swapped halfway through a request would lose the
registrations made before the swap.

---

## Decision

Each port's Rust trait has a generated PHP twin interface. Binding a port to a PHP object is the override flag; a
stateful port may only be bound to PHP before any PHP runs in the request.

### Rules

1. A port is a Rust trait with a **PHP twin**: a PHP interface with the same methods, generated from the trait by
   `#[php_port]`.
2. A **PHP adapter** of a port is a Rust struct that implements the trait by calling a PHP object that implements the
   twin. Values are converted at that boundary only.
3. Binding a port to a PHP adapter is the override flag. The container records, per key, whether the current binding
   is PHP.
4. Resolving a Rust binding never enters PHP. Hot ports cache their resolved adapter, and the cache is dropped when
   the binding changes.
5. A **stateful port** (a store, such as `HookStore`) is bound to PHP in the app's configuration, `php.bindings`,
   installed at request start before any PHP runs.
6. Rebinding a stateful port from PHP after it was used in the request is refused, with an error naming the port.
7. Stateless ports may be rebound from PHP at any point.

### What this looks like

✅ **Correct**

```rust
// crates/speedyspec-container-php/tests/php.rs: the twin is generated from the trait.
#[php_port(name = "SpeedySpec\\Testing\\Greeter")]
pub trait Greeter {
    fn greet(&self, name: String) -> String;
}
```

```toml
# A stateful port overridden in config, before any PHP runs.
[app.blog.php]
bindings = { "SpeedySpec\\WpHooks\\HookStore" = "App\\HookStore" }
```

❌ **Not permitted**

```php
// A stateful port rebound from PHP after it was used. It is refused with:
// "... holds state and was already used in this request; bind it in app.<id>.php.bindings instead"
add_filter( 'init', 'my_init' );
\SpeedySpec\Container\Container::getInstance()->bind( 'SpeedySpec\WpHooks\HookStore', fn () => new MyStore() );
```

### Patterns

- **Adapter:** the PHP adapters wrap PHP objects behind Rust traits.
- **Strategy:** a port's implementation is chosen by its binding.

---

## Consequences

### Positive

1. A PHP twin cannot drift from its trait, because it is generated.
2. With no override, a call stays in Rust.
3. A PHP store sees every registration from the start of a request.

### Negative

1. Every port needs a twin, and every PHP adapter converts values at the boundary.
   **Mitigation:** the macro generates both; conversion only happens when PHP is bound.
2. A plugin cannot swap a store halfway through a request.
   **Mitigation:** the refusal names the port and points to `php.bindings`.

### Trade-offs

**We prioritise:**
- Rust-only calls when nothing is overridden.
- A store that sees a whole request.

**We accept:**
- A twin per port.
- Stateful overrides only in configuration.

---

## Fleet impact

- **speedyspec-php-runtime:** reads `php.bindings` from each worker's spec and installs them at request start.
- **speedyspec-server:** passes `app.<id>.php.bindings` to the runtime in the worker spec.
- **speedyspec-cli:** wires its ports through the same container, so a PHP binding overrides a port there the same
  way. Unverified whether the CLI exposes `php.bindings` configuration.
- **PHP apps and plugins:** a PHP override of a store must be declared in configuration and implement the twin (for
  the hook store, `SpeedySpec\WpHooks\HookStorage`).

---

## Enforcement

### Automated

- `cargo test -p speedyspec-container-php --test php` runs `stateful_port_bound_after_use_is_refused`,
  `stateful_port_bound_before_use_is_allowed` and `override_is_gone_after_a_fatal_error`. A failure exits 1.
- `cargo test -p speedyspec-wp-hooks-php --test php_store` checks a PHP hook store bound through the twin.
- Both run in the server's CI `php` job. Not confirmed seen failing.

### Review checklist

- [ ] Every new port trait carries `#[php_port]`.
- [ ] PHP adapters convert values only at the boundary.
- [ ] A new stateful port is registered as stateful, so late rebinding is refused.
- [ ] Rust callers of a port with no PHP binding never enter PHP.

---

## Exceptions

None.

---

## Revision history

- **2026-11-18:** decided, in the PHP foundation design.
- **2026-11-19:** shipped in SpeedySpec/speedyspec-server#4, with the twin generated by the `#[php_port]` macro.
- **2026-11-21:** recorded as an ADR, and rewritten in this structure.
