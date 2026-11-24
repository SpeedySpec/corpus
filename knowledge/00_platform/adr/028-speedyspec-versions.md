# ADR-028: SpeedySpec 1 stays WordPress; later majors replace it

**Status:** Accepted
**Date:** 2026-11-23
**Deciders:** both: the project owner (the three majors, the runtime carrying the promise, SpeedySpec 1 following
WordPress for good, old APIs kept as adapters over the new ones); the designer (the deprecation rule, the plugin
header, the departures register)
**Exploration:** [SpeedySpec versions](../../../explorations/2026-11-23-speedyspec-versions.md)
**Related:** [ADR-005](005-core-hooks-stay.md), [ADR-014](014-port-wordpress-to-rust-and-delete-its-php.md), [ADR-017](017-php-twins-and-the-override-flag.md), [ADR-021](021-recorded-compatibility.md)

---

## Context

SpeedySpec ports WordPress to Rust one area at a time, and adds APIs of its own: blocks in place of widgets, ports a
plugin can rebind, a detached admin. Two goals pull against each other. Site owners want to put SpeedySpec on top of an
existing WordPress install and have it work, with every plugin they already run. The project wants plugins moved onto
SpeedySpec's own APIs, so that the old WordPress systems can eventually go.

Each component already has its own SemVer: the CLI is on 1.x, the PHP runtime on 1.x, the server on 0.x. None of
those numbers says what a plugin can rely on.

### Why this matters

Without a line between "compatible" and "replacing", every port is a judgement call. One area keeps a WordPress quirk,
the next drops it, and a plugin author cannot tell which SpeedySpec release will break their code. Removing an old API
without warning breaks sites on an ordinary upgrade. Keeping every old API for good means the new APIs never replace
anything.

---

## Decision

SpeedySpec has numbered majors, and the major of the PHP runtime is the SpeedySpec version. SpeedySpec 1 keeps as much
compatibility with WordPress and PHP as it can, follows WordPress's releases for good, and never breaks a plugin in a
1.x upgrade. SpeedySpec 2 and later majors remove what an earlier major deprecated, and may change tables.

### Rules

1. **The SpeedySpec version is the runtime's major.** SpeedySpec 1 is `speedyspec-php-runtime` 1.x; SpeedySpec 2 is
   2.x. Pre-releases and development builds of a major follow the rules of the major they lead to.
2. **The promise covers what plugins call in PHP**: WordPress's functions, classes, hooks, globals, constants, options
   and tables, as the runtime serves them, and SpeedySpec's own PHP APIs. The WordPress ports in the server
   workspace and their copies in the CLI count as runtime code for this rule.
3. **The rest is outside the promise.** The CLI, the server, the static admin, the admin REST API (`wp-admin/v1`),
   the blog API and WordPress's own `wp/v2` REST API each keep their own versioning, and none of them promises
   WordPress compatibility. `register_rest_route()` is a PHP API and stays inside the promise; the shapes of core's
   `wp/v2` routes do not.
4. **The admin REST API and the blog API honour the promise as consumers.** Each declares the SpeedySpec majors it
   supports. Supporting a major means a plugin's use of that major's PHP APIs works through it: a menu page, a
   setting, a meta box or a REST route registered the WordPress way is picked up and acted on. Their own endpoints and
   response shapes are not part of the promise.
5. **No upgrade within a major breaks a plugin.** A minor adds; a patch fixes. A change that brings behaviour closer
   to WordPress's is a fix.
6. **SpeedySpec 1 follows WordPress for good.** Each WordPress release is recorded
   ([ADR-021](021-recorded-compatibility.md)) and ported in a 1.x minor, including WordPress's own schema upgrades and
   deprecations. This has no end date; only a later decision can give it one.
7. **Every departure from WordPress in SpeedySpec 1 is written down**, with how a plugin or site works with it: in the
   departures register of the [versions exploration](../../../explorations/2026-11-23-speedyspec-versions.md), and in
   the area's `breaks.json` where a replay covers it. A departure that is not in the register is a bug.
8. **An old API that a new one replaces stays, as an Adapter over the new one.** It behaves as WordPress's did; the new
   API is the only implementation underneath. In SpeedySpec 1, `WP_Widget` builds blocks, but a plugin's widget still
   renders, saves and shows its form as before.
9. **Nothing is removed in a major unless the previous major deprecated it.** The deprecation ships in a minor, names
   the replacement, and fires WordPress's deprecation hooks (`deprecated_function_run` and the rest).
10. **SpeedySpec 1 keeps WordPress's tables as WordPress has them.** It may add tables of its own, but changes
    WordPress's only as WordPress does. A later major that changes them ships the migration, and says before it runs
    that it cannot be undone except from a backup.
11. **Plugins declare the majors they support** with a `Requires SpeedySpec:` header holding a SemVer range, such as
    `1 || 2`. A plugin without the header is a WordPress plugin, and is treated as `1`.

### What this looks like

✅ **Correct**

```php
<?php
/**
 * Plugin Name: Opening Hours
 * Requires SpeedySpec: 1 || 2
 */

// SpeedySpec 1: the WordPress widget API still works, as an adapter that builds blocks.
// SpeedySpec 1.x deprecates it, naming the block API as the replacement; SpeedySpec 2 removes it.
// The plugin supports both majors by registering a block, and the widget only where it still exists.
add_action( 'init', 'opening_hours_register_block' );

if ( class_exists( 'WP_Widget' ) ) {
	add_action( 'widgets_init', 'opening_hours_register_widget' );
}
```

❌ **Not permitted**

```text
SpeedySpec 1.4 stops calling a WP_Widget subclass's form(), because the block editor no longer needs it.
SpeedySpec 2.0 removes get_page_by_title() without a 1.x deprecation that named WP_Query as its replacement.
SpeedySpec 1.6 adds a column to wp_posts that WordPress does not have.
```

### Patterns

- **Published Interface** (Fowler): the PHP APIs of a SpeedySpec major are published, so they change only by the
  deprecation rule; the admin REST API and the blog API are not, so they change on their own SemVer.
- **Adapter** (Gang of Four): each old WordPress API kept in a major translates to the new SpeedySpec API that replaces
  it. Dependencies point inward: the adapter depends on the new API, never the reverse.
- **Strangler Fig** (Fowler): each major removes the adapters the previous one deprecated, so the new APIs replace the
  old ones a major at a time.
- **Golden Master**: following WordPress is proved by re-recording each WordPress release
  ([ADR-021](021-recorded-compatibility.md)).

---

## Consequences

### Positive

1. A site owner can upgrade within SpeedySpec 1 without checking plugins, and can put SpeedySpec 1 on an existing
   WordPress install.
2. A plugin author knows which APIs will go, and what replaces them, a full major before they go.
3. The new APIs are the only implementations, so an old API cannot drift from them.
4. The admin and the blog API can change their own endpoints without waiting for a SpeedySpec major.

### Negative

1. SpeedySpec 1 has to be ported forward for every WordPress release, for good, alongside the later majors.
   **Mitigation:** each release is re-recorded and replayed, so the work is driven by recorded differences; the
   project owner expects LLM-assisted porting to keep this small.
2. Some WordPress behaviour cannot be kept even in SpeedySpec 1.
   **Mitigation:** each departure is in the register with its workaround, and the project owner approves the area's
   design.
3. An old API kept as an adapter must reproduce WordPress's behaviour over an API shaped differently.
   **Mitigation:** the adapter is held to WordPress's recorded behaviour like any other port.
4. Plugins without the header are assumed to need SpeedySpec 1, so they cannot run on SpeedySpec 2 until updated.
   **Mitigation:** this is intended; the plugin's author, or the project, updates and declares it.

### Trade-offs

**We prioritise:**
- No broken plugins within a major.
- A new API as the single implementation of each behaviour.

**We accept:**
- Maintaining SpeedySpec 1 against WordPress with no end date.
- Adapters for old APIs, written to WordPress's behaviour.

---

## Fleet impact

- **speedyspec-php-runtime:** its major is the SpeedySpec version. It reads `Requires SpeedySpec:` and serves the PHP
  APIs of its major.
- **speedyspec-server:** its WordPress port crates follow the runtime's versioning rules (rule 2). The server's own
  version is outside the promise.
- **speedyspec-cli:** outside the promise, but its copies of the port crates follow the runtime's rules
  ([ADR-023](023-shared-crates-in-the-server-workspace.md)).
- **Admin REST API and blog API:** declare the SpeedySpec majors they support, and honour those majors' PHP APIs.
- **Static admin:** outside the promise; front-end framework majors move only its own version.
- **SpeedySpec Guide:** shows, for each symbol, the major that deprecates it and the major that removes it.
- **WordPress plugins and themes:** keep working on SpeedySpec 1; declare `Requires SpeedySpec:` to run on later
  majors.

---

## Enforcement

### Automated

- The replays of [ADR-021](021-recorded-compatibility.md) hold each ported area to WordPress's recorded behaviour. A
  difference not listed in the area's `breaks.json` fails `cargo test`.
- Nothing yet checks that `breaks.json` and the departures register agree, that a removal had a deprecation in the
  previous major, or that the runtime reads `Requires SpeedySpec:`. Those are review only.

### Review checklist

- [ ] A change to runtime code that a plugin can observe is a fix, an addition, or listed in the departures register.
- [ ] Every new departure has a workaround in the register and, where a replay covers it, a `breaks.json` entry.
- [ ] A removal in a new major names the 1.x (or earlier-major) minor that deprecated it, and its replacement.
- [ ] An old API replaced by a new one is an adapter over it, not a second implementation.
- [ ] A table change lands only in a new major, with its migration.
- [ ] Each WordPress release has been recorded and ported into SpeedySpec 1.

---

## Exceptions

A departure in SpeedySpec 1 that WordPress compatibility cannot survive, such as one forced by security, needs the
project owner's approval and an entry in the register before it ships. Ending SpeedySpec 1's tracking of WordPress
needs a new decision record. How long SpeedySpec 2 and later majors get fixes once the next major ships is decided per
major by the project owner, from how hard the fixes are and how many installs still run it.

---

## Revision history

- **2026-11-23:** decided, in the SpeedySpec versions discussion.
- **2026-11-23:** the project owner placed `wp/v2` outside the promise, and set how later majors' fix windows are
  decided.
