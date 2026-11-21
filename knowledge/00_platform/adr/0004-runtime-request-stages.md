# 0004. The runtime answers requests in four stages before PHP

- **Status:** Accepted, 2026-11-19
- **Decided by:** the designer

## Context

With the features in the runtime (0002), each request needs a fixed order in which they get to answer. Stock
WordPress has two such points: Wordfence loads through `auto_prepend_file`, and the page cache through
`advanced-cache.php` when `WP_CACHE` is set.

## Decision

The runtime handles each request in this order, all in Rust before any PHP:

1. **Request rules** (the firewall), where `auto_prepend_file` would run.
2. **Response cache** lookup, where `advanced-cache.php` would run.
3. **Routing:** route, redirect, or not found.
4. **Native responders** (sitemaps, `robots.txt`, feeds), or failing those, the PHP script.

Which apps run which stages:
- Request rules run for every PHP app: the blog, the front-end API and the admin API.
- The response cache and native responders run for the blog and the front-end API only. The admin API is never cached.
- Login protection attaches to the admin API's sign-in endpoint, where authentication happens in SpeedySpec.

## Consequences

- A cache hit, a blocked request, a redirect, a sitemap, `robots.txt` or a feed served from Rust never starts the PHP
  script.
- A stage sees only the request's FastCGI params, never its body, because PHP may still need the body. Rules over the
  body wait for a decision on buffering it.
- The first stage was built as `speedyspec-native-stage` (`StageChain`) with the firewall (speedyspec-server #20).

## Patterns

- **Chain of Responsibility:** the stages, asked in order; the first to respond wins.
