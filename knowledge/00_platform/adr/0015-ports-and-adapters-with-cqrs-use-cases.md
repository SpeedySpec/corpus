# 0015. Ports and Adapters, with every WordPress operation a CQRS use case

- **Status:** Accepted, 2026-11-18. Crate naming superseded in part on 2026-11-19 (see Consequences).
- **Decided by:** the project owner

## Context

Every WordPress area will be ported the same way (0014), so the shape of a port has to be settled once. The project
owner asked for Clean Architecture and Clean Code, with the Repository and Service patterns, and for every WordPress
function and method to be a use case.

## Decision

- **Inbound ports are the operations:** one per WordPress function and per public method, each a CQRS **command**
  (it changes state) or **query** (it only reads). A command returns nothing, or only what WordPress's signature needs.
  A query has no side effects. An operation that is necessarily both is a command carrying a result, and each such
  exception is listed in the area's design.
- **Handlers are the use cases**, one per inbound port. A handler depends only on ports, which the container injects
  (0018). It never reads globals.
- **Outbound ports are what the core needs**, named for the capability as a noun (`HookStore`). Repositories are
  outbound ports, split into read and write sides where they differ.
- **Domain services are pure shared logic**, named for what they compute (`PriorityOrder`). They have no adapters.
- **Driven adapters** are named technology first (`MemoryHookStore`). **Driving adapters** are named entry point first
  (`WpHookFunctions`).
- **Requests and results are values:** every command and query is a struct of its inputs, with a result type.
- Ports and their implementations never share a crate.
- No name ends in `Service`, `Contract`, `Interface` or `Manager`.

## Alternatives considered

- **DDD with layer packages** (`-domain`, `-app`, `-infra-*`, `-legacy`). Rejected at the time by the project owner.
- **Clean Architecture rings with "services".** Replaced by Ports and Adapters naming, which names each kind of
  service precisely.

## Consequences

- The Hook API's crates follow this: `speedyspec-wp-hooks` holds the core and ports, `speedyspec-wp-hooks-memory` and
  `speedyspec-wp-hooks-php` hold adapters.
- **Superseded in part:** this record also said no crate is named `-domain`, `-app` or `-infra`. The native plugin
  features design (0001 to 0013) brought layer packages back: `-domain`, `-app`, `-infra-*`, `-presentation` and
  `-legacy`, never `-core`. Later crates, such as the firewall's, use those layers. Keeping contracts apart from
  their implementations still holds.

## Patterns

- **Hexagonal Architecture:** inbound and outbound ports, driving and driven adapters.
- **Command Query Responsibility Segregation:** the operations.
- **Repository:** the outbound stores.
