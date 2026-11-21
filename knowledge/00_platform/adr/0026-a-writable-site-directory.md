# 0026. Each PHP app has a writable site directory over its read-only releases

- **Status:** Accepted, 2026-11-18
- **Decided by:** the project owner; the layering by the designer

## Context

Releases are read-only and switched atomically. WordPress writes uploads, plugins, themes and its configuration under
its own directory, and secrets must never be in a release.

## Decision

- Each PHP app has its own writable directory (`site_dir`) that survives every release switch. It holds the site's
  configuration, uploads, and installed plugins and themes.
- The app sees the release with the site directory's `wp-content` layered over it: the worker sets `WP_CONTENT_DIR`
  and `WP_CONTENT_URL` to the site directory.
- Static files are served from the site directory first, then the release.
- Configuration can come as environment variables, so secrets and database credentials stay out of releases.

## Consequences

- A release switch never loses uploads or installed plugins.
- Installing plugins through the admin is left to later work; until then they are put in place by hand or by the CLI.

## Patterns

- **Layered file system**, as an overlay of the site directory on the release.
