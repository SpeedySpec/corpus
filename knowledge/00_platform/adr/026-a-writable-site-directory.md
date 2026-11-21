# ADR-026: Each PHP app has a writable site directory over its read-only releases

**Status:** Accepted
**Date:** 2026-11-18
**Deciders:** both: the project owner (a writable site directory); the designer (the layering)
**Exploration:** [PHP foundation](../speedyspec-server/explorations/2026-11-18-php-foundation.md)
**Related:** [ADR-022](022-fastcgi-over-a-unix-socket.md), [ADR-025](025-supervised-php-worker-pools.md)

---

## Context

Releases are read-only and switched atomically ([ADR-025](025-supervised-php-worker-pools.md)). WordPress, though,
writes into its own directory: uploads, installed plugins and themes, and its configuration in `wp-config.php`.
Secrets such as database credentials must never be part of a release, because releases are built and shipped
elsewhere.

### Why this matters

If WordPress wrote into the release, every release switch would lose uploads and installed plugins. If secrets were
in the release, anyone with the release artifact would have them, and every app would share one set of credentials.

---

## Decision

Each PHP app has its own writable site directory, `site_dir`, that survives every release switch. The app sees the
release with the site directory's `wp-content` layered over it, and takes its configuration and secrets from the site
directory and the environment.

### Rules

1. Each PHP app has its own `site_dir`, required in managed mode. It survives every release switch.
2. The site directory holds the site's configuration, uploads, and installed plugins and themes.
3. Workers define `WP_CONTENT_DIR` as `site_dir/wp-content` before the app's code runs.
4. Static files under `wp-content` are served from the site directory first, then from the release.
5. Only `site_dir/wp-content` is served from the site directory, and no dotfile is ever served: it holds secrets.
6. Configuration and secrets come from the environment (`site_dir/.env`, then `env_file`, then `env`), never from the
   release.

### What this looks like

✅ **Correct**

```rust
// crates/speedyspec-php-runtime/src/worker.rs: run before every request's script.
if let Some(site) = &spec.site_dir {
    let content = site.join("wp-content");
    code.push_str(&format!(
        "if (!defined('WP_CONTENT_DIR')) {{ define('WP_CONTENT_DIR', {}); }}\n",
        quote(&content.display().to_string())
    ));
}
```

❌ **Not permitted**

```php
// A wp-config.php shipped inside the release, with credentials in it.
define( 'DB_PASSWORD', '<the real password>' );
```

### Patterns

- **Layered file system:** the site directory is overlaid on the release.

---

## Consequences

### Positive

1. A release switch never loses uploads or installed plugins.
2. Secrets stay out of releases, and each app can have its own credentials.
3. Releases stay read-only and can be switched atomically.

### Negative

1. Installing plugins through the admin is not supported yet.
   **Mitigation:** plugins are put in place by hand or by the CLI until later work adds it.
2. Each app's site directory must be provisioned and backed up by the operator.
   **Mitigation:** none yet.
3. Credentials are isolated between apps only as far as their workers are: pools of one OS user can read each other's
   process environment.
   **Mitigation:** none yet; run apps as different users where it matters.

### Trade-offs

**We prioritise:**
- Read-only, atomic releases.
- Secrets kept out of releases.

**We accept:**
- A second directory per app to manage.
- No plugin installs from the admin for now.

---

## Fleet impact

- **speedyspec-server:** requires `site_dir` per PHP app in managed mode, serves `wp-content` from it first, and
  merges each app's environment before its pool starts.
- **speedyspec-php-runtime:** defines `WP_CONTENT_DIR` and allows scripts from the release or from
  `site_dir/wp-content`.
- **PHP apps (admin API, front-end API, blog):** must read secrets from the environment, not from their release.
  Whether their releases use the site directory yet is unverified.
- **speedyspec-cli:** places plugins and themes into the site directory until the admin can install them.
- **WordPress plugins and themes:** live in `site_dir/wp-content`, not in the release.

---

## Enforcement

### Automated

- `cargo test -p speedyspec-server --features test-hooks --test php` runs `static_file_from_site_dir_then_release`,
  `private_files_are_not_served`, `php_in_site_wp_content_runs` and
  `worker_secrets_stay_out_of_argv_and_the_server_environment_stays_out`. A failure fails `cargo test`. It runs in the
  server's CI `php` job. Not confirmed seen failing.
- Nothing checks that a release ships no secrets. That is enforced by review only.

### Review checklist

- [ ] Nothing the app writes goes into the release directory.
- [ ] Secrets come from the environment or the site directory, never from the release.
- [ ] New served paths from the site directory stay inside `wp-content` and skip dotfiles.

---

## Exceptions

None.

---

## Revision history

- **2026-11-18:** decided, in the PHP foundation design.
- **2026-11-19:** shipped in SpeedySpec/speedyspec-server#4. As built, the worker defines only `WP_CONTENT_DIR`;
  WordPress derives `WP_CONTENT_URL` from `siteurl`, because building it from the request's `Host` would let any
  client choose it.
- **2026-11-19:** the PHP runtime and admin gate design added per-app environments merged from `site_dir/.env`,
  `env_file` and `env`, and passed workers their spec on stdin so secrets stay out of the process table
  (SpeedySpec/speedyspec-server#6).
- **2026-11-21:** recorded as an ADR, and rewritten in this structure.
