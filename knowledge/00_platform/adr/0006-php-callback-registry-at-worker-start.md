# 0006. A registry of PHP callbacks, built when a worker starts

- **Status:** Accepted, 2026-11-19
- **Decided by:** the project owner

## Context

PHP callbacks normally exist for one request only: PHP registers them as it loads plugins and the theme. A request
answered in Rust never loads them, so it would never see them, and core hooks must keep working (0005).

## Decision

Each runtime worker loads its app's plugins and theme once at startup and records which hooks have PHP callbacks.

- A native path (redirect, sitemap, `robots.txt`, feed, cache-key check, firewall rule) checks the registry for every
  hook it fires. If none has a PHP callback, the request stays in Rust. If any does, the request goes to PHP, and the
  PHP path fires the hook as usual.
- The registry is rebuilt whenever a worker is replaced. Plugin activation, deactivation and new releases all recycle
  workers.
- A callback registered only on some requests is still recorded.

## Consequences

- The registry may send a request to PHP more often than needed, but never less. Correctness wins over speed.
- Rules added by PHP plugins, such as request rules for the firewall, wait for this registry.
- Rust callbacks that a feature registers must land in the hook store's worker-lifetime partition, at worker start.

## Patterns

- **Registry**, built once per worker and read on every native path.
