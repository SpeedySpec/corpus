# 0001. Plugin features are behaviour of the ported WordPress APIs

- **Status:** Accepted, 2026-11-19
- **Decided by:** the designer, accepted by the project owner

## Context

SpeedySpec ports WordPress to Rust and embeds PHP for what remains. The aim is to keep as little PHP in flight per
request as possible. Four kinds of popular plugins run PHP on nearly every request:

- page caching (WP Rocket, W3 Total Cache, WP Super Cache);
- firewall and login security (Wordfence, Solid Security);
- redirects, `robots.txt`, sitemaps and feeds (Redirection, the sitemap parts of Yoast and Rank Math);
- image optimization (Smush, EWWW, Imagify).

Almost everything these plugins do already has a home in WordPress core. Plugins mostly extend core subsystems, and
core already calls its cache-clearing functions on every change.

## Decision

Each feature becomes behaviour of the WordPress API it extends (rewrite, sitemaps, robots, feeds, options, posts,
terms, users, auth, media), added when that API is ported to Rust. Each feature runs in Rust and never enters the PHP
interpreter.

Not in scope:
- compatibility with the original plugins' own PHP APIs, classes, hooks or option names; importing their settings is
  optional;
- CDN integration, CSS and JavaScript minification or bundling, critical CSS, database cleanup, malware scanning of
  files, two-factor authentication, SEO content scoring.

## Alternatives considered

- **Separate Rust plugin packages** that hook into the ported APIs from outside. Lost because each would reimplement
  invalidation and image generation beside the APIs that already know when data changes.

## Consequences

- Nothing is written twice: cache invalidation and image generation fall out of the ports.
- A feature can't arrive before the API it extends (see 0003).
- A feature that reads WordPress's tables directly, instead of through a ported API, is a defect.
- If a write path can change content without the cache hearing about it through core's own functions, the design has
  failed.

## Patterns

- **Ports and adapters:** every feature sits behind a contract. Contracts and implementations live in separate
  packages, so an implementation can be replaced partly or wholly.
- **Repository:** every storage-backed contract has an in-memory implementation for tests and a real one.
