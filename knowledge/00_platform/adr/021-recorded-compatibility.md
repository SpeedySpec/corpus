# ADR-021: Recorded compatibility proves a port before WordPress's code is deleted

**Status:** Accepted
**Date:** 2026-11-18
**Deciders:** both: the project owner (prove by recording WordPress's outputs); the designer (the format and harness)
**Exploration:** [PHP foundation](../speedyspec-server/explorations/2026-11-18-php-foundation.md)
**Related:** [ADR-014](014-port-wordpress-to-rust-and-delete-its-php.md), [ADR-020](020-rust-owns-hook-state.md), [ADR-024](024-static-php-with-every-wordpress-extension.md)

---

## Context

Once an area is ported, WordPress's implementation of it is deleted
([ADR-014](014-port-wordpress-to-rust-and-delete-its-php.md)). Before that, something has to prove that the port
behaves exactly as WordPress did. After it, something has to keep proving it, with no WordPress code left to compare
against. WordPress's own unit tests help, but they do not pin every return value, callback order and global.

### Why this matters

Without a recorded baseline, a port could change behaviour that plugins rely on, and nobody would know until a site
broke. After deletion, the original would be gone, so the question "what did WordPress do here?" could no longer be
answered locally.

---

## Decision

Each area has a catalogue of scenarios, recorded once against WordPress's original code and committed. A replayer
holds the port to that recording, exactly, before and after the original is deleted.

### Rules

1. Each area has a **catalogue** of scenarios, as JSON: sequences of calls to WordPress functions and methods.
   Callbacks are named fixtures, so a scenario holds no PHP code.
2. A **recorder** runs each scenario against WordPress's original implementation in the same static PHP, and commits
   what came back: return values, callback calls in order, and the observable globals after each step.
3. Each fixture names the WordPress version it came from. Re-recording fetches that exact upstream release, checked
   by SHA-256.
4. A **replayer** runs the scenarios against the port and compares exactly. Any difference fails, unless the area's
   design lists it as a documented break in its `breaks.json`.
5. WordPress's implementation of an area is deleted only after the replayer and WordPress's own tests for the area
   pass.
6. The fixtures and the replayer stay as the area's compatibility tests after deletion.

### What this looks like

✅ **Correct**

```jsonc
// compat/hooks/scenarios.json: one scenario, with a named callback fixture.
{ "name": "add_filter_returns_true",
  "steps": [ { "call": "add_filter",
               "args": [ "f", { "callback": "append", "kind": "closure", "label": "a", "with": { "suffix": "-a" } } ] } ] }
```

```jsonc
// compat/hooks/breaks.json: a difference is allowed only when listed, with its reason.
{ "scenario": "wp_filter_replace_wholesale",
  "why": "Replacing $wp_filter wholesale: the global becomes a plain PHP array, no longer the proxy over the Rust hook store, which does not see it." }
```

❌ **Not permitted**

```text
Editing compat/hooks/wordpress-7.1.5.json by hand so the port passes,
or deleting an area's WordPress code before its replay is clean.
```

### Patterns

- **Golden Master** (characterization testing): WordPress's recorded output is the reference.

---

## Consequences

### Positive

1. A port is proven against WordPress's real behaviour, not against a reading of its code.
2. The proof survives the deletion of WordPress's code.
3. Every allowed difference is written down, with its reason.

### Negative

1. Re-recording needs the original WordPress release and the static PHP.
   **Mitigation:** the recorder fetches the pinned release (`compat/<area>/source.json`) and checks its hash.
2. A catalogue only covers what it lists; untested behaviour can still differ.
   **Mitigation:** WordPress's own tests for the area also run in the embedded PHP and stay after deletion.

### Trade-offs

**We prioritise:**
- Exact, observable compatibility.
- Proof that outlives the original code.

**We accept:**
- Writing and maintaining a catalogue per area.
- Re-recording when the pinned WordPress version moves.

---

## Fleet impact

- **speedyspec-server:** holds `speedyspec-compat` (recorder and replayer) and `compat/<area>/` for every ported
  area: hooks, options, object cache and wpdb so far, all recorded from WordPress 7.1.5.
- **speedyspec-cli:** carries copies of `speedyspec-compat` and `compat/hooks`
  ([ADR-023](023-shared-crates-in-the-server-workspace.md)).
- **wordpress-develop:** an area's code is deleted only after its replay passes.
- **Future area ports:** must add a catalogue, a fixture and a `breaks.json` before any deletion.

---

## Enforcement

### Automated

- `cargo test -p speedyspec-compat --test format` checks the replayer itself, including
  `listed_break_is_tolerated_and_unlisted_is_not`. A failure fails `cargo test`.
- `cargo test -p speedyspec-wp-hooks-php --test php` runs the hook replay (`replayed_scenarios_match_wordpress`, 89
  scenarios). Each mismatch is printed, and the test binary exits 1.
- The other ported areas replay the same way: `cargo test -p speedyspec-wp-options-php --test compat`,
  `cargo test -p speedyspec-wp-object-cache-php --test php` and `cargo test -p speedyspec-wp-db-php --test replay`.
- All run in the server's CI `php` job. Not confirmed seen failing.
- `speedyspec-compat record <area> --wordpress <version>` re-records a fixture. It is a tool, not a check.

### Review checklist

- [ ] The area has a catalogue, a fixture naming its WordPress version, and a `breaks.json`.
- [ ] Every entry in `breaks.json` has a reason and appears in the area's design.
- [ ] Fixtures change only by re-recording, never by hand.
- [ ] Replay and WordPress's own tests pass before the deletion is proposed.

---

## Exceptions

Documented breaks, listed in the area's design and its `breaks.json`. The project owner approves the design.

---

## Revision history

- **2026-11-18:** decided, in the PHP foundation design.
- **2026-11-19:** shipped in SpeedySpec/speedyspec-server#4, with WordPress 7.1.5's hook behaviour recorded for 89
  scenarios and 182 upstream hook tests passing in the embedded PHP.
- **2026-11-21:** recorded as an ADR, and rewritten in this structure.
