# ADR-019: Command and query buses, with a direct path for hot operations

**Status:** Accepted
**Date:** 2026-11-18
**Deciders:** both: the project owner (CQRS); the designer (the direct path)
**Exploration:** [PHP foundation](../speedyspec-server/explorations/2026-11-18-php-foundation.md)
**Related:** [ADR-015](015-ports-and-adapters-with-cqrs-use-cases.md), [ADR-018](018-a-service-container-in-rust.md), [ADR-020](020-rust-owns-hook-state.md)

---

## Context

Every WordPress operation is a command or a query ([ADR-015](015-ports-and-adapters-with-cqrs-use-cases.md)).
Something has to route each one to its handler, and add cross-cutting work such as logging and transactions. Some
operations are very hot: `apply_filters` and `get_option` run thousands of times in one request.

### Why this matters

Without a bus, every caller would find its own handler and repeat logging and transaction code. With only a bus,
every hot call would pay for a container lookup and a middleware chain, and the Rust port would be slower than the
PHP it replaces.

---

## Decision

`speedyspec-cqrs` provides command and query buses with ordered middleware. A driving adapter on a hot path may
resolve a handler once and call it directly, skipping the bus and its middleware.

### Rules

1. `Command` and `Query` are traits, each with a result type. `CommandHandler` and `QueryHandler` handle them.
2. `CommandBus` dispatches commands and `QueryBus` answers queries. A command cannot be sent through the query bus.
3. Both buses run ordered middleware. Handlers are resolved from the container by the message's key.
4. **Direct dispatch:** a driving adapter may resolve a handler once (`HandlerRef`, `QueryRef`) and call it with no
   middleware.
5. Middleware applies to bus dispatch only. Work that must always run belongs in the handler, not in middleware.
6. PHP can dispatch commands and queries, and register handlers, through native classes. A PHP handler is a PHP
   adapter like any other.

### What this looks like

✅ **Correct**

```rust
// crates/speedyspec-wp-options-php/src/native.rs: get_option() resolves its handler once.
pub(crate) struct Handlers {
    get_option: QueryRef<GetOption>,
    add: HandlerRef<AddOption>,
    // ...
}
// ...
h(|h| ask(&h.get_option, GetOption { name: n, default }))?
```

❌ **Not permitted**

```rust
// crates/speedyspec-cqrs/tests/ui/query_bus_refuses_command.rs: does not compile.
let bus = QueryBus::new(Rc::new(Container::new()));
let _ = bus.ask(AddPost); // AddPost is a Command, not a Query
```

### Patterns

- **Command:** each operation is a value with its inputs.
- **Mediator:** the buses route messages to handlers.
- **Chain of Responsibility:** the middleware.

---

## Consequences

### Positive

1. Cross-cutting work is written once, as middleware.
2. Hot paths pay one lookup, not one per call.
3. A handler rebound in the container is followed by `HandlerRef`, so overrides still work on the direct path.

### Negative

1. Middleware such as logging and transactions does not run on the direct path.
   **Mitigation:** none yet; hot paths are chosen knowing this.
2. Two ways to call a handler.
   **Mitigation:** the direct path is for driving adapters on hot paths only.

### Trade-offs

**We prioritise:**
- Speed on hot paths.
- One place for cross-cutting work.

**We accept:**
- Middleware that the direct path skips.

---

## Fleet impact

- **speedyspec-php-runtime:** native WordPress functions use direct dispatch. Which bus middleware ship beyond
  logging is left to the builder.
- **speedyspec-server:** holds the crate. The server binary uses it only where its own areas need it. Unverified.
- **speedyspec-cli:** `speedyspec-cqrs` is among the copied crates ([ADR-023](023-shared-crates-in-the-server-workspace.md)).
- **PHP apps and plugins:** may dispatch and handle through the native classes in `speedyspec-cqrs-php`.

---

## Enforcement

### Automated

- `cargo test -p speedyspec-cqrs` runs `middleware_runs_in_order`, `direct_dispatch_skips_middleware`,
  `handler_ref_follows_rebinding` and the compile-fail test `query_bus_refuses_command`. A failure fails
  `cargo test`. It runs in the server's CI `test` job. Not seen failing.
- `cargo test -p speedyspec-cqrs-php --test php` checks PHP dispatch and PHP handlers, in the CI `php` job.
- Nothing checks which paths use the direct path. That is enforced by review only.

### Review checklist

- [ ] The direct path is used only by driving adapters on hot paths.
- [ ] Nothing that must always run is put in middleware.
- [ ] Commands go through the command bus and queries through the query bus.

---

## Exceptions

None.

---

## Revision history

- **2026-11-18:** decided, in the PHP foundation design.
- **2026-11-19:** shipped in SpeedySpec/speedyspec-server#4.
- **2026-11-21:** recorded as an ADR, and rewritten in this structure.
