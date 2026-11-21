# 0005. WordPress core's hooks stay; new hooks only for what core lacks

- **Status:** Accepted, 2026-11-19
- **Decided by:** the designer

## Context

PHP plugins and themes hook into the areas the native features take over. They must keep working.

## Decision

- Core's hook names, and the order they fire in, stay unchanged.
- New hooks exist only for what core lacks (redirect rules, cache tags, request rules), under the project's prefix.
- Settings use the project's own option names, read through the ported options API.
- Compatibility tests record WordPress's real output for the core parts (canonical redirects, sitemaps, `robots.txt`,
  feeds, image sub-sizes) and replay it against the ports. Plugin-derived extras get their own tests.

## Consequences

- Each feature names the core hooks it keeps (see 0010 to 0013).
- A request that fires a hook with a PHP callback must go to PHP (see 0006).

## Patterns

- **Observer**, through the Hook API.
