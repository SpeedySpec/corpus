# ADR-018: A service container in Rust, with Laravel's container's functionality

**Status:** Accepted
**Date:** 2026-11-18
**Deciders:** both: the project owner (a Rust container with Laravel's container's functionality); the designer (the PHP-facing details)
**Exploration:** [PHP foundation](../speedyspec-server/explorations/2026-11-18-php-foundation.md)
**Related:** [ADR-015](015-ports-and-adapters-with-cqrs-use-cases.md), [ADR-017](017-php-twins-and-the-override-flag.md), [ADR-019](019-command-and-query-buses.md), [ADR-023](023-shared-crates-in-the-server-workspace.md)

---

## Context

Every port needs an adapter wired to it ([ADR-015](015-ports-and-adapters-with-cqrs-use-cases.md)), from Rust and
from PHP alike. A PHP worker lives for many requests, but PHP objects belong to one request. Anything PHP registers
must vanish when its request ends, while what Rust registered at start-up must stay. Laravel's service container is
a well-known design with the features needed.

### Why this matters

Without one container, each area would wire its own adapters, and a PHP override could not reach them. Without two
lifetimes, a PHP object kept by the container would outlive its request and point at freed memory, or leak one
request's state into the next.

---

## Decision

`speedyspec-container` is a Rust service container with the functionality of Laravel's, under its method names
where they fit. It keeps worker-lifetime and request-lifetime bindings apart, and PHP reaches it through one native
class.

### Rules

1. The container offers `bind`, `singleton`, `scoped`, `instance`, `make`, `alias`, `extend`, contextual binding,
   tags, resolving callbacks, `call`, `flush` and autowiring, under Laravel's names where they fit.
2. Nothing promises Laravel compatibility. Laravel is not supported.
3. Keys are strings: a port's name or a PHP class. Rust callers use typed helpers (`bind_port`, `make_port`).
4. **Worker lifetime:** bindings, aliases, tags and callbacks registered from Rust, and Rust adapters.
5. **Request lifetime:** anything PHP made or registered. All of it is cleared at PHP request shutdown, which puts
   every port back on its worker-lifetime binding.
6. PHP reaches the container only through the native `SpeedySpec\Container\Container`. There are no `app()` or
   `resolve()` global helpers.
7. Handlers get their ports injected. The container is used as a locator only at the edges: driving adapters and
   start-up.

### What this looks like

✅ **Correct**

```rust
// crates/speedyspec-container/tests/container.rs: PHP's override ends with its request.
c.singleton("port", |_, _| Ok(string("rust")));
{
    let _request = c.registering(Lifetime::Request);
    c.bind("port", |_, _| Ok(string("php override")));
}
c.end_request();
assert_eq!(text(&c.make("port").unwrap()), "rust", "the worker-lifetime binding is back");
```

❌ **Not permitted**

```php
// A Laravel-style global helper, or a handler pulling its own dependencies.
function app( $key = null ) { return \SpeedySpec\Container\Container::getInstance()->make( $key ); }
```

### Patterns

- **Dependency Injection:** handlers receive their ports.
- **Service Locator:** at the edges only, in driving adapters and start-up.
- **Registry:** tags.
- **Decorator:** `extend`.

---

## Consequences

### Positive

1. One place wires every port, for Rust and for PHP.
2. A PHP object never outlives the PHP request that made it.
3. The method names are familiar to PHP developers.

### Negative

1. Rust callbacks that must last a worker's life have to be registered at worker start, not during a request.
   **Mitigation:** anything registered during a request is request lifetime by design; start-up code is the place
   for worker-lifetime wiring.
2. Similar names may suggest Laravel code will run against it.
   **Mitigation:** the design and docs say Laravel is not supported.

### Trade-offs

**We prioritise:**
- Safe request boundaries for PHP objects.
- One wiring point for every port.

**We accept:**
- Two lifetimes to reason about.
- No global helpers for PHP code.

---

## Fleet impact

- **speedyspec-php-runtime:** builds the container at worker start and ends its request half at every request
  shutdown.
- **speedyspec-server:** holds the crate in its workspace; the server binary itself does not embed PHP.
- **speedyspec-cli:** carries a copy of `speedyspec-container` and `speedyspec-container-php` and wires its ports the
  same way ([ADR-023](023-shared-crates-in-the-server-workspace.md)).
- **PHP apps and plugins:** use `SpeedySpec\Container\Container::getInstance()`; there is no `app()`.

---

## Enforcement

### Automated

- `cargo test -p speedyspec-container` covers the container's behaviours, including
  `end_request_removes_request_lifetime_everything` and `scoped_is_shared_until_end_request`. It runs in the
  server's CI `test` job; a failure fails `cargo test`. Not seen failing.
- `cargo test -p speedyspec-container-php --test php` checks the container from PHP, including
  `override_is_gone_after_a_fatal_error`. It runs in the CI `php` job.
- Nothing checks that handlers avoid the locator. That is enforced by review only.

### Review checklist

- [ ] Handlers take their ports as constructor arguments, not from the container.
- [ ] Worker-lifetime Rust wiring happens at start-up.
- [ ] No `app()` or `resolve()` helper is added.
- [ ] Anything PHP creates is request lifetime.

---

## Exceptions

None.

---

## Revision history

- **2026-11-18:** decided, in the PHP foundation design.
- **2026-11-19:** shipped in SpeedySpec/speedyspec-server#4.
- **2026-11-19:** copied into the CLI in SpeedySpec/speedyspec-cli#11.
- **2026-11-21:** recorded as an ADR, and rewritten in this structure.
