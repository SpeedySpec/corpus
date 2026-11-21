# 0010. The firewall runs first; login protection lives in the auth API

- **Status:** Accepted, 2026-11-19
- **Decided by:** the designer

## Context

Wordfence and Solid Security filter requests before WordPress loads, and protect sign-in. Core already has the
`authenticate` filter, `wp_login_failed`, application passwords, XML-RPC, `?author=N` lookups and the REST users
endpoint.

## Decision

- **Request rules** run as stage 1 (0004): IP allow and block lists, rate limits, and patterns over the path, query
  string and body.
- **Login protection** lives in the ported auth API: `wp_login_failed` counts failures and the `authenticate` filter
  enforces lockouts.
- Switches close user enumeration and XML-RPC.
- Counters and bans sit behind a store contract (0007).
- Kept or added: `authenticate`, `wp_login_failed`, `xmlrpc_enabled`, `rest_endpoints`, and a hook for adding request
  rules.

## Consequences

Built in speedyspec-server #20. That work settled what the design left open:

- The admin API runs with an admin scope that has no rate limit and no IP bans, so the owner can't lock themselves
  out of their own admin.
- Counters and bans key IPv6 clients by their /64 network.
- Windows, bans and lockouts are capped at thirty days.
- If the firewall can't do its job (a store fails, settings can't be read, no client address), the request goes
  through and a warning is logged.
- Until a ported auth API exists, lockouts are enforced at the request stage, for `POST`s to the sign-in paths.
- Deferred: body patterns (0004), rules added by PHP plugins (0006), and `authenticate` enforcement.

## Patterns

- **Chain of Responsibility:** the rules.
- **Specification:** IP sets and pattern rules.
- **Strategy:** the store backends.
- **Observer:** the hook callbacks.
