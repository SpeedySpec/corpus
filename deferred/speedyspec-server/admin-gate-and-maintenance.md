# Admin gate and maintenance

`speedyspec-server-gate`, `speedyspec-server-presentation`, `speedyspec-server` (`php/calls.rs`). From the PHP
runtime and admin gate plan (#6).

## "**" in an access rule does not match a newline

- **Where:** `speedyspec-server-gate`, `rules.rs`, `glob_regex`.
- **What happens:** `**` compiles to `.*` without the `s` flag, so it stops at a newline. A path containing an encoded
  newline (`%0A`) does not match a rule it would otherwise match, and falls through to the next rule or the default.
- **Suggested fix:** compile with `(?s)`, or refuse control characters when the path is normalized.

## Public "maintenance" entries can never be reached

- **Where:** `speedyspec-server-presentation`, `service.rs` (maintenance is answered before the access check);
  `speedyspec-server-gate`, `rules.rs` (`PublicWhen::Maintenance`).
- **What happens:** a rule with `public = "maintenance"` makes a path public only while its app is in maintenance. But
  any request to an app in planned maintenance gets the maintenance page before the gate is asked, so the rule never
  takes effect.
- **Suggested fix:** decide what these entries are for. Either check them before answering maintenance (so a page can
  stay reachable during it), or remove the `maintenance` condition.

## The spec and the code disagree on a malformed 200 session answer

- **Where:** `docs/superpowers/specs/2026-11-19-speedyspec-php-runtime-and-admin-gate-design.md` (the Session
  contract); `speedyspec-server`, `php/calls.rs`.
- **What happens:** the spec says a `200` session answer with a malformed body counts as signed in with no pages. The
  code treats a `200` without a valid positive user id as signed out, which is safer. Only a valid user with a
  malformed page list gets no pages.
- **Suggested fix:** update the spec to match the code.
