# Exploration: The SpeedySpec Guide, a tested reference for the WordPress API on SpeedySpec

**Status:** built in part
**Created:** 2026-11-20
**Author:** Claude, for the project owner
**Context:** A static documentation site that says, for every WordPress symbol, whether it still works on SpeedySpec, backed by citations and examples run in CI.
**Approved by:** not recorded
**Socialized with:** not socialized

---

## Problem Statement

SpeedySpec's runtime replaces WordPress's PHP API one area at a time with Rust. The Hook API is done and Options is next. Each replacement can change behaviour, and every 1.x release is part of the API contract. Two groups need to know what changed:

- **Porters**, plugin and theme developers who arrive with a WordPress name and ask whether their code still works, and what to change if not.
- **Builders**, the project owner and later contributors, who need to see how each part is put together and how to change it.

developer.wordpress.org does not answer this. It is organised by file, and it renders PHPDoc, which can be wrong.

## Current State

- No SpeedySpec documentation site existed for the API.
- speedyspec-server's `speedyspec-compat` harness records WordPress behaviour as scenarios and breaks, in `compat/<area>/`. The fixtures are recorded against WordPress 7.1.5.
- The fork's `packages/wp-*` split in SpeedySpec/wordpress-develop already marks out likely seams (`wp-plugin` for Hooks, `wp-option` for Options).
- The runtime's worker boots its native areas in `speedyspec-php-runtime`. The request-scoped hook registry there is private to the binary.
- No `speedyspec-php-runtime-v*` tag exists, so every SpeedySpec symbol is unreleased.
- Other Astro sites in the project use Bun and ship a Docker image.

## Proposed Solution

A new repository, SpeedySpec/speedyspec-guide: an Astro static site plus a build toolchain that extracts facts, checks every claim and runs every example.

### Architecture

```
 sources.json (WordPress 7.1.5 tarball + sha256, server commit, fork commit)
        |
        v
 extract --> fork --> assign --> graph --> history --> examples --> reflect
   (PHP      (same/    (symbols   (calls,    (since     (Rust       (runtime
   parser)   modified/  to seams)  hooks,     from tags)  runner)     API)
             removed)              cycles)
        \___________________________|_____________________________/
                                    v
                         validate (Specification rules)
                                    v
                         generated/*.json + seams/*/seam.yaml
                         + content/*.mdx + examples/*.php
                                    v
                    Astro site: /wordpress/<seam>/  /speedyspec/<seam>/
                    Mermaid -> SVG at build time, Pagefind search
```

- Hexagonal toolchain in TypeScript: a plain domain package, ports in `@speedyspec-guide/contracts`, one package per adapter, an app package of use cases, and a CLI as the composition root.
- Pipes and Filters: each stage reads the last stage's files in `generated/`, so any stage runs alone.
- The examples runner, `speedyspec-guide-examples`, is a Rust crate in the guide repo. It links the server's crates and static PHP from a pinned checkout through a symlink, and only executes: it runs PHP files and prints what each printed, returned and threw.

### Data Model

- Symbol ids: `function:<name>`, `class:<Name>`, `method:<Class>::<name>`, `global:<name>`, `filter:<hook>`, `action:<hook>`. Dynamic hooks keep their template, such as `filter:pre_option_{$option}`.
- Edges: `calls`, `reads`, `writes`, `fires`, `runs`, `matches`, `alias`, `undone-by`, each with a citation of the call site.
- Citations: a compat scenario, a documented break, a SpeedySpec test, or source lines (`wp:` for WordPress 7.1.5, `ss:` for SpeedySpec) with a stored hash of the lines.
- Verdicts: works as WordPress, works with differences, breaks, new in SpeedySpec. For unported symbols the verdict is computed: same code, modified, or removed.
- `seam.yaml`: name, job, ported or not, files, symbols, roles, edges, and later packages, crates and dependency states.
- Each example is a PHP file plus `.expect.json` holding `sides`, `output`, `return` and `error`.

### Interfaces

- Ports: `SymbolSource`, `ReleaseSource`, `SeamRepository`, `EvidenceSource`, `ReleaseHistory`, `ExampleSource`, `ExampleRunner`.
- Runner CLI: `run --side <wordpress|runtime> --manifest <file>` and `reflect --mode <baseline|runtime>`.
- Validation rules: `EverySymbolInOneSeam`, `CitationExists`, `CitedLinesUnchanged`, `PortedSeamVerified`, `WordPressPagesCiteWordPress`, `VersionHasTag`, `ExampleHasRun`, `PortedSeamCovered`, and later `CitesItsScenario`.

## Design Decisions

1. **The guide holds prose and examples and borrows the compat harness's data.** The project owner chose this. CI checks out a pinned speedyspec-server; a nightly run against its `main` opens an issue when an example fails.
   *Trade-off:* the guide must follow a server it may not edit.
2. **Organise by seam, not by file.** A seam is a boundary where behaviour can be swapped. Each page shows where its symbol sits in the seam.
   *Trade-off:* every symbol must be placed by hand or by rule, and the build fails on conflicts.
3. **Separate WordPress and SpeedySpec sections.** The project owner wanted the WordPress API readable on its own. WordPress pages keep a verdict pill and a short "On SpeedySpec" box.
   *Trade-off:* two pages per ported symbol.
4. **Every claim cites evidence, and the build checks it.** Ported seams need scenarios, breaks or tests; source citations alone fail. Unported seams may cite code and say "read from source". PHPDoc is never a source.
   *Trade-off:* writing is slow, and moved lines break the build until re-cited.
5. **Versions are computed, never typed.** WordPress "since" comes from parsing every upstream release tag. SpeedySpec "added in" is recorded per pinned server commit; walking every runtime tag was dropped while planning because no tag exists and each build is costly.
   *Trade-off:* a release the pin skips is dated to the next recorded one.
6. **The guide has its own runner crate.** `speedyspec-compat`'s download and loading code is private, found while planning.
   *Trade-off:* duplicated loading logic; a test fails when the runtime boots an area the runner does not.
7. **Plain Astro with content collections, not Starlight.** The designer judged the seam rail and relationship panel too custom for Starlight's layout.
   *Trade-off:* more layout code to own.
8. **Cycles are found and shown in red with the word "circular".** Tarjan's algorithm over symbols and seams, including paths through hooks. Only WordPress's own hook registrations count.
   *Trade-off:* plugin callbacks can close cycles the guide cannot see.

## Implementation Phases

1. **Toolchain (plan 1).** Release download with sha256 check, PHP extractor, seam assignment, dependency graph and cycles, fork comparison, WordPress history, evidence and validation, CLI, CI.
   *Delivers:* `generated/*.json` for WordPress 7.1.5. **Shipped** on `main`, confirmed in the repository's commit history.
2. **Examples runner (plan 2).** The Rust crate, Reflection of the runtime's API, runtime history, example validation, nightly run against server `main`.
   *Delivers:* examples run on both sides in CI. **Shipped**, confirmed the same way.
3. **Site (plan 3).** Slugs, data loader, views, Mermaid, components, both sections, coverage page, search, Docker and CI.
   *Delivers:* the static site. **Shipped**, confirmed the same way.
4. **Hooks content (plan 4).** An evidence index and `CitesItsScenario` rule, the full Hooks seam map, pages and examples for every Hook API function, `WP_Hook` method and global, porting guide, architecture page, then coverage enforced.
   *Delivers:* the verified tier proven on Hooks. **Not shipped:** no evidence-index commit, and `requirePortedCoverage` is still false.
5. **Every other symbol (plan 5).** Map every WordPress symbol to a seam, give the runtime's container and plugin-registry API their own seams, record each dependency's state. Not started.

## Fleet Impact

- **SpeedySpec/speedyspec-guide:** new repository holding the toolchain, runner and site. Confirmed with `gh`.
- **speedyspec-server:** read only, pinned by commit. The guide sent three requests: optional snippets in `speedyspec-compat`; treat `speedyspec-php`'s public items as used by the guide; make the request-scoped hook registry public.
- **SpeedySpec/wordpress-develop:** its `packages/` tree is read to compare the fork with WordPress. Plan 3 also adds a local preview entry for the site. No other change.
- **Upstream WordPress/wordpress-develop:** release tags fetched read only.
- Hosting is undecided. Unverified beyond these.

## Open Questions

1. **Which seam owns a hook fired from several seams?** — the spec said the first in load order; the extractor cannot see load order, so planning made it an explicit listing in `seam.yaml`. Needs the project owner's confirmation.
2. **Will the runner move into `speedyspec-compat`?** — depends on the server session adding snippets.
3. **Will the request-scoped hook registry become public?** — until then the runner carries its own copy; without it a callback outlived its request and crashed the next example.
4. **Which Hooks symbols have no scenario?** — plan 4's evidence index will list them; they need SpeedySpec tests cited instead.
5. **Where does the site run?** — hosting was left for later.
6. **How should Pagefind index names and aliases, and how are multi-job packages split into seams?** — left to the builder.

## Success Criteria

- [x] WordPress 7.1.5 parses with no errors — README records 16,943 symbols, 63,254 edges and 555 unresolved calls.
- [x] Cycles found, including through hooks — README records 243 circular groups holding 1,330 symbols; one closes only through a hook-name pattern and is marked possible.
- [x] An example runs on both WordPress and the runtime — `hooks/add_filter/change-a-value` passed on both on its first run.
- [x] The runtime's own API is listed by Reflection — 181 symbols, all unreleased.
- [x] The site builds from real data — 61 pages in 4 seconds, 57 indexed for search.
- [x] Fatal errors, throws and missing returns become an example's result without stopping the run — runner tests.
- [ ] Every Hooks symbol has verified pages on both sides — plan 4, not done.
- [ ] Ported-seam coverage enforced — `requirePortedCoverage` still false.

---

## Non-Goals

- **Per-release copies of the site** — one current version, with a history on each symbol.
- **Hosting and deployment** — the repo produces a static build and Docker image only.
- **Changes to speedyspec-server** — another session owns it.
- **Rendering PHPDoc as prose** — comments can be wrong.

## Alternatives Considered

### Option: The guide brings its own extractor and example runner, separate from compat
- **Pros:** self-contained.
- **Cons:** duplicates `speedyspec-compat`; the two would drift.
- **Verdict:** Rejected.

### Option: Each source repository publishes a doc bundle per release
- **Pros:** breakage caught at the source.
- **Cons:** needs changes in repositories another session owns; nothing shows until all publish.
- **Verdict:** Rejected.

### Option: Starlight
- **Pros:** ready-made docs layout.
- **Cons:** gets in the way of the seam rail and relationship panel.
- **Verdict:** Rejected by the designer.

## Dependencies

- speedyspec-server's compat scenarios and breaks, its crates, and its static PHP build.
- The WordPress 7.1.5 release tarball and upstream release tags.
- The Hook API port, the first ported seam, which proves the verified tier.

## References

- Source: spec `2026-11-20-speedyspec-guide-design.md`; plans `2026-11-20-speedyspec-guide-toolchain.md`, `2026-11-20-speedyspec-guide-examples-runner.md`, `2026-11-20-speedyspec-guide-site.md`, `2026-11-20-speedyspec-guide-hooks-content.md`.
- [SpeedySpec/speedyspec-guide](https://github.com/SpeedySpec/speedyspec-guide)
- [SpeedySpec/speedyspec-server](https://github.com/SpeedySpec/speedyspec-server)
- [WordPress/wordpress-develop](https://github.com/WordPress/wordpress-develop)
