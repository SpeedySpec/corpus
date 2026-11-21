# ADR-023: The engine crates live in the server workspace and are copied into the CLI

**Status:** Accepted
**Date:** 2026-11-18
**Deciders:** both: the project owner (copying into the CLI); the designer (the location and the drift check)
**Exploration:** [PHP foundation](../speedyspec-server/explorations/2026-11-18-php-foundation.md)
**Related:** [ADR-018](018-a-service-container-in-rust.md), [ADR-020](020-rust-owns-hook-state.md), [ADR-021](021-recorded-compatibility.md), [ADR-024](024-static-php-with-every-wordpress-extension.md)

---

## Context

The container, the buses and the Hook API are needed by two programs: the server side and the CLI. The CLI runs
WordPress PHP plugins, and those keep calling `add_filter` after WordPress's PHP hook API is deleted. The CLI already
has a crate named `speedyspec-hooks`, the hook contract for its own plugins. There was no shared repository for engine
code.

### Why this matters

If the CLI lacked these crates, WordPress plugins it runs would stop working once WordPress's PHP hook API was gone.
If the two copies drifted silently, a plugin could behave differently under the CLI than on a site.

---

## Decision

The engine crates live in the `speedyspec-server` workspace, with no dependency on the server. The CLI keeps copies,
records their source, and fails CI if a copy no longer matches what was recorded.

### Rules

1. The engine crates live in the `speedyspec-server` workspace and do not depend on the server crates.
2. The server repository is the source. A change lands there first, then is copied into the CLI.
3. The CLI's copies are recorded in `copied-from.json`, with the source commit and a content hash per copied path.
4. A CI check in the CLI fails when a copied path's content no longer matches its recorded hash.
5. The WordPress hook area is `speedyspec-wp-hooks` in both repositories, apart from the CLI's own `speedyspec-hooks`.
6. The CLI's PHP build uses the same extension list as the server's
   ([ADR-024](024-static-php-with-every-wordpress-extension.md)).

### What this looks like

✅ **Correct**

```jsonc
// speedyspec-cli/copied-from.json (shortened)
{
  "source": "packages/speedyspec-server",
  "commit": "c6c991bf7bb4adeec4150298c7ff25fc14fbcccd",
  "paths": {
    "crates/speedyspec-container": "e57873bc0ea6c7fb12257905f5eaac6f22e81f928d27c3dd0931e53eff1f2f16",
    "crates/speedyspec-wp-hooks": "d7d21fe0aca910ddb88bf6ffa024bf9598669d08188ec3afeebdf6c13e26abce"
  }
}
```

❌ **Not permitted**

```text
Fixing a bug in speedyspec-cli/crates/speedyspec-wp-hooks directly, without landing it in
speedyspec-server first and re-copying.
```

### Patterns

- **Shared Kernel:** one engine shared by two programs, kept in step by a check rather than a shared repository.

---

## Consequences

### Positive

1. WordPress plugins run by the CLI keep their hook API.
2. Both programs wire ports the same way.
3. No new repository or cross-repository CI was needed.

### Negative

1. The copies fall behind whenever the server's crates change.
   **Mitigation:** none automatic. The check only compares each copy with the hash recorded when it was copied; it
   does not compare with the server's current code. Re-syncing is the CLI's own work.
2. Every engine change needs a second step in the CLI.
   **Mitigation:** none yet. A shared `speedyspec-engine` repository was rejected for now.

### Trade-offs

**We prioritise:**
- No new repository until the CLI needs more of the engine.
- Drift inside the CLI is caught, not silent.

**We accept:**
- Copies that lag the server.
- A manual re-copy for every change.

---

## Fleet impact

- **speedyspec-server:** owns the engine crates and must keep them free of server dependencies.
- **speedyspec-cli:** carries copies of `speedyspec-compat`, `speedyspec-container`, `speedyspec-container-php`,
  `speedyspec-cqrs`, `speedyspec-php`, `speedyspec-php-ext`, `speedyspec-php-port-macros`, the three
  `speedyspec-wp-hooks` crates, `compat/hooks` and `tests/php`. It must re-copy after server changes and update
  `copied-from.json`.
- **speedyspec-php-runtime:** not affected; it builds from the server workspace directly.

---

## Enforcement

### Automated

- In `SpeedySpec/speedyspec-cli`, `scripts/check-copies.sh` hashes each path listed in `copied-from.json` and exits 1,
  naming the path, if a hash differs or a path is missing. CI's "copies match speedyspec-server" job runs it on pushes to
  `main` and on every pull request. In the recorded CI runs the step has always passed; it has not been seen
  failing.
- Nothing checks that the copies match the server's current code, or that the engine crates stay free of server
  dependencies. Those are enforced by review only.

### Review checklist

- [ ] A change to a copied crate lands in `speedyspec-server` first.
- [ ] A re-copy updates `copied-from.json` with the new commit and hashes.
- [ ] Engine crates gain no dependency on server crates.

---

## Exceptions

None.

---

## Revision history

- **2026-11-18:** decided, in the PHP foundation design.
- **2026-11-19:** the copies and the drift check shipped in SpeedySpec/speedyspec-cli#11.
- **2026-11-21:** recorded as an ADR, and rewritten in this structure.
