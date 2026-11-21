# 0022. The server talks to PHP workers over FastCGI on a Unix socket

- **Status:** Accepted, 2026-11-18
- **Decided by:** the designer

## Context

The server runs PHP apps in supervised worker processes, the way php-fpm does. It needs a protocol to them.

## Decision

FastCGI 1.0 over a Unix socket.

## Alternatives considered

- **A private protocol over a socketpair**, like the keeper's. Smaller, but nothing else could speak it.
- **Running PHP in the server process.** Rejected: one fatal error or leak would take the server down.

## Consequences

- A worker can be tested and debugged on its own with any FastCGI tool.
- FastCGI stayed the only channel between server and runtime when the runtime became its own component (0025): the
  gate's questions and health checks are ordinary app endpoints over the same path.

## Patterns

- **Gateway:** the server's FastCGI client.
