# 0002. Native features run in the runtime's Rust side, before PHP

- **Status:** Accepted, 2026-11-19
- **Decided by:** the project owner

## Context

SpeedySpec has two processes. The server (Salvo) holds no PHP and no WordPress knowledge; it knows only app kinds. The
PHP runtime embeds PHP and is reached over FastCGI, with no side channel beside FastCGI. The native features (0001)
need to answer requests before PHP runs.

## Decision

All native features run in the runtime's Rust side, answering before the PHP script starts.

## Alternatives considered

- **A. Split:** HTTP-level parts (response cache, rate limits, IP blocks, request rules) in the server, driven by what
  the app tells it; WordPress-specific parts in the runtime. The fastest path, and it keeps the server rule. Kept as a
  later experiment.
- **B. All in the server:** simplest to reason about, but gives the server database access and WordPress knowledge,
  which overturns the runtime's design.

## Consequences

- The server is untouched.
- Cached and blocked requests still cross FastCGI into the runtime, so a cache hit costs a FastCGI round trip.
- Static admin-site files are served by the server and never reach the runtime, so the firewall doesn't cover them.
  This is accepted here and revisited with approach A.
- A server-level cache or firewall in front of FastCGI is out of scope until that experiment.

## Patterns

- **Hexagonal Architecture:** the runtime is the application core; FastCGI is its only port to the server.
