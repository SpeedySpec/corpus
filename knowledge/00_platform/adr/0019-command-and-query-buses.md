# 0019. Command and query buses, with a direct path for hot operations

- **Status:** Accepted, 2026-11-18
- **Decided by:** the project owner (CQRS); the direct path by the designer

## Context

Use cases are commands and queries (0015). Some, such as `apply_filters`, run thousands of times per request.

## Decision

- `speedyspec-cqrs` has `Command` and `Query` traits with result types, `CommandHandler` and `QueryHandler` traits, and
  `CommandBus` and `QueryBus` with ordered middleware. Handlers are resolved from the container.
- **Direct dispatch:** a driving adapter may resolve a handler once and call it with no middleware. WordPress's hook
  functions do this.
- PHP can dispatch commands and queries, and register handlers, through native classes. A PHP handler is a PHP
  adapter like any other.

## Consequences

- Middleware such as logging and transactions applies to bus dispatch only, not to the direct path.

## Patterns

- **Command**, **Mediator** (the buses), **Chain of Responsibility** (middleware).
