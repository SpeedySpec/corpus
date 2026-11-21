# 0023. The engine crates live in the server workspace and are copied into the CLI

- **Status:** Accepted, 2026-11-18
- **Decided by:** the project owner (copying into the CLI); the location and drift check by the designer

## Context

The container, the buses and the Hook API are needed by the server and by the CLI. The CLI runs WordPress PHP
plugins that keep calling `add_filter` once WordPress's PHP hook API is deleted.

## Decision

- The crates live in the `speedyspec-server` workspace, with no dependency on the server.
- `speedyspec-container`, `speedyspec-wp-hooks`, `-memory` and `-php` are copied into the CLI's workspace and wired
  into its embedded PHP, so the CLI wires its ports the same way.
- The WordPress hook area is `speedyspec-wp-hooks` in both, because the CLI already has `speedyspec-hooks` for its own
  plugins.
- **The server repository is the source.** The CLI's copies record the commit they came from, and a CI check in the CLI
  compares them with it, so drift fails a check instead of passing silently.
- The CLI's PHP build gets the same extension list as the server's (0024).

## Alternatives considered

- **A shared `speedyspec-engine` repository.** Cleaner, but needs a new repository and cross-repository CI, for no gain
  until the CLI moves.

## Consequences

- A change to these crates lands in the server first, then is copied into the CLI.

## Patterns

- **Shared kernel**, kept in step by a check rather than by a shared repository.
