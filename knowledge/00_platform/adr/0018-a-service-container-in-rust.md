# 0018. A service container in Rust, with Laravel's container's functionality

- **Status:** Accepted, 2026-11-18
- **Decided by:** the project owner; the PHP-facing details by the designer

## Context

Every port needs an adapter wired to it (0015), from Rust and from PHP alike, and PHP objects must never outlive their
request.

## Decision

- `speedyspec-container` has the functionality of Laravel's service container, and reuses its method names where they
  fit: `bind`, `singleton`, `scoped`, `instance`, `make`, `alias`, `extend`, contextual binding, tags, resolving
  callbacks, `call`, `flush` and autowiring. Nothing promises Laravel compatibility.
- Keys are strings, such as a port's name or a PHP class. Rust callers use typed helpers.
- **Two lifetimes:**
  - *worker lifetime*: bindings, aliases, tags, callbacks registered from Rust, and Rust adapters;
  - *request lifetime*: anything PHP made or registered. All of it is cleared at PHP request shutdown, which puts
    every port back on its worker-lifetime binding.
- PHP reaches the same container through the native `SpeedySpec\Container\Container`. There are no `app()` or
  `resolve()` helpers.

## Alternatives considered

- **Global `app()` and `resolve()` helpers.** Rejected: WordPress has neither, and they would only be there for
  Laravel.

## Consequences

- A PHP object never outlives the PHP request that made it.
- Rust callbacks that must last a worker's life have to be registered at worker start, not during a request.

## Patterns

- **Service Locator** at the edges only (driving adapters and start-up); handlers get their ports injected.
- **Dependency Injection**, **Registry** (tags), **Decorator** (`extend`).
