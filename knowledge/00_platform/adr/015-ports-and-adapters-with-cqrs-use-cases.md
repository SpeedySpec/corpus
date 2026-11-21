# ADR-015: Ports and Adapters, with every WordPress operation a CQRS use case

**Status:** Accepted, superseded in part by the native plugin features design's layer crates
**Date:** 2026-11-18
**Deciders:** the project owner
**Exploration:** [PHP foundation](../speedyspec-server/explorations/2026-11-18-php-foundation.md)
**Related:** [ADR-001](001-plugin-features-are-behaviour-of-ported-apis.md), [ADR-014](014-port-wordpress-to-rust-and-delete-its-php.md), [ADR-017](017-php-twins-and-the-override-flag.md), [ADR-018](018-a-service-container-in-rust.md), [ADR-019](019-command-and-query-buses.md)

---

## Context

Every WordPress area will be ported the same way ([ADR-014](014-port-wordpress-to-rust-and-delete-its-php.md)), so
the shape of a port had to be settled once, before the first one. The project owner asked for Clean Architecture and
Clean Code, with the Repository and Service patterns, and for every WordPress function and method to be a use case.
WordPress itself has no such shape: functions read and write globals freely.

### Why this matters

Without one shape, each area would invent its own. Adapters could not be swapped, PHP could not override a piece
cleanly ([ADR-017](017-php-twins-and-the-override-flag.md)), and reviewers would have nothing to check a port
against. A vague word such as "service" would hide what each piece does.

---

## Decision

Each WordPress area is a hexagon. Its inbound ports are the WordPress operations, each a CQRS command or query with a
handler. Its outbound ports are what the core needs, implemented by swappable adapters.

### Rules

1. One inbound port per WordPress function and per public method. Each is a **command** (it changes state) or a
   **query** (it only reads).
2. A command returns nothing, or only what WordPress's signature needs. A query has no side effects.
3. An operation that must be both is a command carrying a result. Each such case is listed in the area's design (for
   hooks: `ApplyFilters`, `DoAction` and their variants).
4. One handler per inbound port. A handler depends only on ports, injected by the container. It never reads globals.
5. Outbound ports are named for the capability, as a noun (`HookStore`). Repositories are outbound ports, split into
   read and write sides where they differ.
6. Domain services are pure shared logic, named for what they compute (`PriorityOrder`). They have no adapters.
7. Driven adapters are named technology first (`MemoryHookStore`). Driving adapters are named entry point first
   (`WpHookFunctions`).
8. Every command and query is a struct of its inputs, with a result type.
9. A port and its implementation never share a crate.
10. No type name ends in `Service`, `Contract`, `Interface` or `Manager`.

### What this looks like

✅ **Correct**

```rust
// crates/speedyspec-wp-hooks/src/messages.rs
pub struct AddFilter { pub hook: HookName, pub callback: Callback, pub priority: HookValue, pub accepted_args: i64 }

impl Command for AddFilter {
    type Result = Result<bool, HookError>;
    const KEY: &'static str = "SpeedySpec\\WpHooks\\AddFilterHandler";
}
```

❌ **Not permitted**

```rust
pub struct HookManager;            // a "Manager" suffix
impl HookManager {
    pub fn add_filter(&self, name: &str) {
        let store = GLOBAL_HOOKS.lock(); // a handler reading a global, not an injected port
    }
}
```

### Patterns

- **Hexagonal Architecture (Ports and Adapters):** inbound and outbound ports, driving and driven adapters.
- **Command Query Responsibility Segregation:** each operation is a command or a query.
- **Repository:** the outbound stores, with read and write sides.

---

## Consequences

### Positive

1. Every area has the same shape, so reviewers check each port against the same rules.
2. Any adapter can be replaced, in Rust or in PHP, without touching the core.
3. Handlers are testable against memory adapters, with no PHP and no globals.

### Negative

1. Hook firing does not fit CQRS: callbacks may change state while a filter returns a value.
   **Mitigation:** it is a listed exception, a command carrying a result.
2. Many small types per area: one struct, one result and one handler per operation.
   **Mitigation:** none yet.
3. The crate naming changed after this decision (see Revision history), so older and newer areas differ in layout.
   **Mitigation:** the rule that contracts and implementations never share a crate still holds for both.

### Trade-offs

**We prioritise:**
- One shape for every port.
- Adapters that can be swapped.

**We accept:**
- More types per area.
- Listed CQRS exceptions where WordPress mixes reads and writes.

---

## Fleet impact

- **speedyspec-server:** every area crate follows this. The Hook API is `speedyspec-wp-hooks` (core and ports) with
  `speedyspec-wp-hooks-memory` and `speedyspec-wp-hooks-php` (adapters). Later areas, such as the firewall, use the
  layer crates `-domain`, `-app`, `-infra-*`, `-presentation` and `-legacy`.
- **speedyspec-php-runtime:** wires the adapters at start-up. It must not bypass the ports.
- **speedyspec-cli:** its copied hook crates keep the same shape ([ADR-023](023-shared-crates-in-the-server-workspace.md)).
- **PHP apps, plugins and themes:** see only the WordPress names; the shape is internal.

---

## Enforcement

### Automated

- `cargo test -p speedyspec-cqrs` includes the compile-fail test `query_bus_refuses_command`: sending a command
  through the query bus does not compile, so `cargo test` fails. It runs in the server's CI `test` job. Not seen
  failing.
- Nothing checks the naming rules, the crate split or the ban on globals in handlers. Those are enforced by review
  only.

### Review checklist

- [ ] Each new WordPress function or public method has one command or query, and one handler.
- [ ] Any operation that is both a command and a query is listed in the area's design.
- [ ] Handlers take ports only, and read no globals.
- [ ] Ports and their implementations are in different crates.
- [ ] No type name ends in `Service`, `Contract`, `Interface` or `Manager`.
- [ ] Adapter names lead with the technology (driven) or the entry point (driving).

---

## Exceptions

Operations that must both change state and return a value are commands carrying a result. Each is listed in the
area's design, which the project owner approves.

---

## Revision history

- **2026-11-18:** decided, in the PHP foundation design. The project owner chose this over DDD layer crates and
  Clean Architecture rings. This version also said no crate is named `-domain`, `-app` or `-infra`.
- **2026-11-19:** superseded in part. The native plugin features design
  ([ADR-001](001-plugin-features-are-behaviour-of-ported-apis.md) to
  [ADR-013](013-images-in-rust-on-the-job-runner.md)) brought layer crates back: `-domain`,
  `-app`, `-infra-*`, `-presentation` and `-legacy`, never `-core`. Keeping contracts apart from implementations still
  holds.
- **2026-11-19:** the Hook API shipped in this shape, in SpeedySpec/speedyspec-server#4.
- **2026-11-21:** recorded as an ADR, and rewritten in this structure.
