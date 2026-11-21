# 0014. WordPress is ported to Rust, and each ported area's PHP is deleted

- **Status:** Accepted, 2026-11-18
- **Decided by:** the project owner

## Context

Most of SpeedySpec's WordPress is moving from PHP to Rust; in the end only plugins will be PHP. The move has to happen
one area at a time, while plugins and themes keep calling WordPress's functions by their usual names.

## Decision

- WordPress's APIs are reimplemented in Rust and wired into the PHP engine as native functions and classes, so the
  remaining PHP and plugins call the same names.
- WordPress's global functions stay, as shims over the Rust ports.
- Once an area is ported and proven (0021), WordPress's own PHP implementation of it is deleted. It is not kept as a
  package, a fallback, or anywhere else.
- This is not a compatible library, and Laravel is not supported.

## Alternatives considered

- **Keeping WordPress's implementation as a fallback.** Rejected: two implementations of one behaviour drift, and the
  fallback would keep PHP in flight.

## Consequences

- A WordPress function's behaviour that plugins can observe (callback order, globals, return values) must not change,
  except where an area's design lists a documented break.
- The Hook API was the first area. Its PHP (`functions/plugin.php`, `class-wp-hook.php` and the PHP hook packages) has
  been deleted.

## Patterns

- **Ports and adapters** (0015): the shims are driving adapters over the Rust ports.
