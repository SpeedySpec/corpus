# 0016. Rust runs as a PHP extension compiled into the embedded PHP

- **Status:** Accepted, 2026-11-18
- **Decided by:** the designer, within the project owner's rule that no PHP sits between callers and Rust

## Context

The Rust ports (0014) must be callable from the PHP that remains, and Rust callers must not pay for PHP.

## Decision

- The Rust code is a PHP extension, written in Rust with `ext-php-rs` and compiled into the embedded, static PHP.
  Native functions and classes run Rust directly.
- Rust code calls Rust directly. It never reaches a Rust adapter through PHP, and never converts values to PHP when no
  PHP adapter is plugged in.
- The server's extension is named `speedyspec_wordpress`, so it never collides with the CLI's `speedyspec` module.

## Alternatives considered

- **A PHP library calling Rust through FFI.** Rejected: it puts PHP code between the caller and Rust.

## Consequences

- WordPress code can't run under an external php-fpm: without the extension it has no WordPress API.
- No Windows, no thread-safe PHP, and no threads running PHP.

## Patterns

- **Adapter:** the native functions and classes are driving adapters over the ports.
