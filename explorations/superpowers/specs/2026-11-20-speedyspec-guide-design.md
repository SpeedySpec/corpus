# SpeedySpec Guide design

Date: 2026-11-20
Status: draft for Jacob's review

A new repository, `speedyspec-guide`: an Astro static site that documents the whole WordPress API as SpeedySpec's PHP
runtime provides it. For each API it says what still works as in WordPress, what doesn't and why, what SpeedySpec adds,
and how to use and override each function and class, with tested examples.

## Intent

**Readers.**

1. **Porters:** plugin and theme developers moving their WordPress code to SpeedySpec (Jacob, 2026-11-20). They
   arrive with a WordPress name they already use and one question: does my code still work, and if not, what do I
   change?
2. **Builders:** Jacob and whoever works on SpeedySpec later. They use the guide to understand what has been built,
   how each part is put together, and how to change it (Jacob, 2026-11-20).

**Why.** SpeedySpec's runtime replaces WordPress's PHP API area by area with Rust (the Hook API is done, the Options
API is next). Every replacement can change behaviour, so porters need to know exactly what changed. The API is
stable: every 1.x minor and patch release is part of the contract, so each one must be documented (Jacob).

**What makes it different from developer.wordpress.org.**

- It is organized by **seams and boundaries**, not by file. Functions and classes that work together are grouped and
  documented together, and each page shows how a function relates to the rest of its seam (Jacob).
- **Reference prose states what is known, never what a comment says.** A page is written from verified behaviour or
  from reading the code. PHPDoc is never the source, because it can be wrong (Jacob).
- It is **concise and easy to read, with minimal distractions** (Jacob).
- **Every example is tested in CI** (Jacob).

## Goals

- Every WordPress function, class, method, hook and global gets a page, grouped by seam (Jacob: "full WordPress
  coverage"). So does every function and class SpeedySpec adds.
- Each page has a **verdict**: *works as WordPress*, *works with differences*, *breaks*, or *new in SpeedySpec*. A
  difference or break comes with its reason and what to do instead.
- Each page lists the **hooks** the function fires (name, arguments, order), and **how to override** it: through a
  filter, as a pluggable function, or by swapping a SpeedySpec adapter.
- **Examples for every function and every class method**, on the reference page itself as well as in the guides
  (Jacob). Ported seams also get porting before/after examples.
- **One current version of the docs**, with a history on each symbol: *added in*, *changed in*, *deprecated in*,
  *removed in*, down to the patch release (Jacob: no per-version copies of the site, but always show when something
  was added).
- **Porting guides** per seam: an overview, a checklist of patterns to search your code for, and links into the
  reference.
- **Architecture pages** per seam for builders: the domain, its ports and adapters, the crates and packages that
  implement them, the patterns used and where, and how to change it (which contract to implement, which scenarios
  and tests must still pass, how to record new ones).
- Search by WordPress name.

## Non-goals

- Copies of the site per release (Jacob).
- Hosting and deployment. The repository produces a static build and a Docker image like the other Astro sites. Where
  it runs is decided later.
- Changes to speedyspec-server's code. Another session owns it (see "Requests to speedyspec-server").
- Rendering WordPress's PHPDoc as prose.

## Anti-goals

- A page claiming something that is not backed by evidence or by a cited line of code.
- An example on the site that has not run in CI.
- A "since" or "added in" version typed by hand.
- Documentation of a ported seam that is anything less than verified.
- A layout that buries the verdict, or a page that hides where a function sits in its seam.

## Constraints

- Astro, static output (Jacob).
- Every behaviour claim carries a **citation**, and the build checks every citation (see "Evidence").
- The design principles in `.claude/rules/design-principles.md` apply to the guide's own tooling.
- The WordPress version documented is the one speedyspec-server's compat fixtures are recorded against, today 7.1.5
  (`compat/hooks/source.json`).
- The SpeedySpec API version is the PHP runtime's version. Its tags are `speedyspec-php-runtime-v<version>` (release
  pipeline design, 2026-11-20). There are no tags yet, so until the first release every symbol shows *unreleased*.

## Approaches

Where the facts and the examples live, and what runs the examples:

1. **The guide brings its own tools.** Its own extractor and its own example runner (WordPress in Docker plus the
   runtime). Self-contained, but it duplicates `speedyspec-compat` and the two would drift. Rejected.
2. **Facts live with the code; the guide renders bundles.** Each source repository publishes a doc bundle (inventory,
   verdicts, examples run by its own CI) with every release. Breakage is caught at the source, but it needs changes
   across repositories another session owns, and nothing can be shown until they all publish. Rejected.
3. **The guide holds prose and examples and borrows the compat harness (chosen by Jacob).** Examples sit beside their
   pages. The guide's CI checks out a pinned speedyspec-server and runs every example through `speedyspec-compat`,
   against WordPress and against the runtime. A nightly run against speedyspec-server's `main` catches breakage before
   the pin moves.

The site itself is plain Astro with content collections and Pagefind, not Starlight. The seam rail and relationship
panel are custom enough that Starlight's layout would get in the way (my call).

## Design

### Repository

- `packages/speedyspec-guide` in this checkout, its own git repository, with GitHub remote `SpeedySpec/speedyspec-guide`,
  like the other `packages/speedyspec-*` repositories (my call). Default branch `main`.
- Bun workspaces and TypeScript for the site and the tooling, matching `astro-sites` (my call). The WordPress
  extractor is a PHP script using `nikic/php-parser`, run by the tooling (my call: PHP parses PHP best).
- `sources.json` pins what is documented: the WordPress release (version, URL, sha256, as in `compat/*/source.json`),
  the speedyspec-server commit, and the `packages/` tree's commit in `SpeedySpec/wordpress-develop`.
- A `Dockerfile` like `astro-sites/speedy-spec/Dockerfile`: `bun run build`, then serve `dist/` with static-web-server.

### Information architecture

The **WordPress API** and the **SpeedySpec API** are separate sections. People can read the WordPress API on its own
(Jacob, 2026-11-20).

```
/                                    what SpeedySpec is, the two APIs, start porting

/wordpress/                          the WordPress API, by seam (Hooks, Options, Object Cache, Query, …)
/wordpress/hooks/                    seam page: its job, members by role
/wordpress/hooks/add_filter/         reference page: WordPress's behaviour, with examples
/wordpress/hooks/WP_Hook/            class page, with every method and its examples
/wordpress/options/filter/pre_option_{option}/   a hook's page, under the seam that fires it

/speedyspec/                         the SpeedySpec API, by seam
/speedyspec/hooks/                   what SpeedySpec does with this seam: native or PHP, compatibility table
/speedyspec/hooks/add_filter/        SpeedySpec's page for a WordPress function: verdict, differences, porting examples
/speedyspec/hooks/<new symbol>/      a function or class SpeedySpec adds
/speedyspec/porting/hooks/           porting guide and checklist for the seam
/speedyspec/architecture/hooks/      builder's page: ports, adapters, crates, patterns, how to change it
```

- **The audience is still people using SpeedySpec** (Jacob, 2026-11-20). The split keeps the WordPress API readable
  on its own, without SpeedySpec's new API mixed in. It doesn't hide SpeedySpec from WordPress pages.
- **WordPress pages describe the WordPress API as it is**, verified against WordPress 7.1.5 or read from its source.
  Because their readers use SpeedySpec, each one keeps the verdict pill in its head and a short **"On SpeedySpec"**
  box: the verdict, one line per difference, and a link to the SpeedySpec page (my call). Their examples run on
  WordPress, and on the runtime too when the verdict is *works as WordPress*.
- **SpeedySpec pages** hold everything SpeedySpec-specific: verdicts, differences and breaks, porting examples, new
  functions and classes, override through adapters, and architecture. A SpeedySpec page for a WordPress function
  links back to its WordPress page instead of repeating it.
- Both sections use the same seams, so the seam rail matches across them. A seam SpeedySpec adds that has no
  WordPress counterpart exists only under `/speedyspec/`.

### Seams

- A **seam** is a boundary where behaviour can be swapped: what WordPress's `packages/wp-*` split already marks out
  (`wp-plugin` → Hooks, `wp-option` → Options, `wp-object-cache` → Object Cache, …), refined where one package holds
  more than one job.
- Each seam is described in `seams/<seam>/seam.yaml`: its name, one sentence on its job, the WordPress packages and
  SpeedySpec crates that implement it, whether it is **ported** (native in the runtime) or **unported**, and its
  members grouped by **role** (for Hooks: register, run, remove, inspect, classes and state).
- **Relationships** are typed edges between members, in `seam.yaml`: *writes*, *reads*, *read by*, *undone by*,
  *alias of*, *fires*, *calls*. Each edge carries a citation like any other claim. The relationship panel on each page
  is drawn from them.
- The build fails if any symbol in the inventory is in no seam, or in more than one.
- **Dependencies between seams are shown everywhere** (Jacob, 2026-11-20), so it is easy to see which seams a
  function has to be unwound from before it can be ported:
  - The extractor builds a **call graph** from 7.1.5's code: every call from one symbol to another, with the call
    site's lines. A call to a symbol in another seam is a **cross-seam dependency**. Reads and writes of another
    seam's globals count too. These are computed, and each cites its call site, so they need no hand-written evidence.
  - **Seam pages** add a "Depends on" list grouped by the other seam (for Options: Multisite › `is_multisite()`,
    Formatting › `sanitize_option()`, Load › `wp_installing()`, …) and a "Depended on by" list. The seam's Mermaid
    diagram shows the other seams as outside nodes, with an arrow per dependency.
  - **Function and method pages** add a "calls into" chip per other seam, listing the symbols called.
  - **SpeedySpec seam and architecture pages** mark each dependency's state for a ported or in-progress seam: still
    calls WordPress's PHP, behind an outbound port with a PHP adapter, or answered in Rust. That is the list of what
    is left to unwind, as the options port design lists its outbound ports.
  - **Circular dependencies have a red background** (Jacob, 2026-11-20), wherever they appear: rows in "Depends on"
    and "Depended on by", "calls into" chips, and nodes and arrows in the Mermaid diagrams (a `cycle` class with a
    red fill). Cycles are found in two graphs, as strongly connected components (Tarjan's algorithm):
    - **between seams:** Options depends on Multisite, and Multisite on Options;
    - **between symbols:** function `a` calls `b`, which calls back into `a`, across seams or within one.

    A seam page lists each cycle it is part of as a path (Options → Multisite → Options), with the call sites that
    close it, because a cycle can't be unwound one side at a time: one of its calls has to go behind a port first.
    The red has text alongside it ("circular") so the meaning doesn't depend on colour alone (my call).
  - Calls the extractor can't resolve statically (`call_user_func` with a variable, `$obj->$method()`) are listed
    as *unresolved* on the page, never dropped.
  - **Filters and actions are part of the graph** (Jacob, 2026-11-20), so cycles through hooks are found too. Two
    more kinds of edge:
    - **fires:** a function calls `apply_filters*` or `do_action*` with a hook's name (a dynamic name is matched as
      a pattern, so `update_option_{$option}` links to every literal `update_option_blogname`);
    - **runs:** WordPress's own code registers a callback on a hook (`add_filter( 'the_content', 'wpautop' )` in
      `default-filters.php`, and every other `add_filter`/`add_action` whose hook name and callback resolve
      statically), so the hook runs that callback.

    A cycle can then go function → hook → callback → hook → … back to the start. This covers a callback that fires
    the hook it's on, and two hooks whose callbacks fire each other. Hook edges are drawn dashed in Mermaid, so a
    cycle through hooks reads differently from a direct call, and red like any other cycle when they close one.
    Callbacks that plugins and themes add can't be known in advance, so only WordPress's own registrations count. A
    page says so next to its hook cycles.
- **A hook belongs to the seam that fires it** (my call): `pre_option_{$option}` is in Options, `the_content` in
  Posts. The Hooks seam holds the Hook API itself (`add_filter`, `apply_filters`, `WP_Hook`, `$wp_filter`, …), not
  every hook in WordPress. A hook fired from more than one seam belongs to the one that fires it first in WordPress's
  load order, and the others link to it.

### Pages

The layout Jacob approved in the mockups (`.superpowers/brainstorm/…/examples-v3.html`). The mockups showed both APIs
on one page. With the split, a WordPress page has the rail, head, relationship panel, signature, examples, behaviour,
fires, override (filters and pluggable functions) and history, with WordPress's provenance. The SpeedySpec page for
the same function has the verdict, differences, porting examples, native/PHP, adapter override and SpeedySpec history.

- **Seam rail** on the left: the whole seam, grouped by role, with the current symbol highlighted. Class pages list
  their methods under the class.
- **Head:** breadcrumb (seam › role), name, verdict pill, and one line of provenance: the tier (verified against
  WordPress 7.1.5 / read from source), *added in*, and whether it is native (Rust) or PHP.
- **Relationship panel:** the symbol's typed edges.
- **Signature**, exact, from the inventory.
- **Examples:** each with a heading saying what it is for, marked as having run in CI. Ported seams add porting
  before/after examples beside the difference they illustrate.
- **Behaviour:** short statements, each with its citation chip.
- **Differs from WordPress** (when it does): what, why (from `breaks.json`), what to do instead.
- **Fires:** hooks, in order, with arguments, each linking to the hook's page.
- **Override:** how to change the behaviour: filter, pluggable function, or adapter (linking to the architecture
  page).
- **History.**
- Class pages: a short description, then every method in the same shape, each method with its own examples.

**Diagrams are Mermaid** (Jacob, 2026-11-20):

- **Seam pages** open with a Mermaid flowchart of the seam: its members and shared state as nodes, its typed edges
  as labelled arrows (the diagram Jacob liked in mockup B). It is generated from the edges in `seam.yaml`, so it shows
  exactly what the citations back.
- **Architecture pages** use Mermaid for the domain, its ports and its adapters, and for the crates and packages that
  implement them. The crate and package graph is generated from the inventory; the rest is written by hand in MDX.
- Function and class pages keep the compact relationship panel (typed chips), which is quicker to scan than a
  diagram (my call).
- Mermaid is rendered to inline SVG at build time (`rehype-mermaid`), so pages ship no diagram JavaScript and the
  diagrams follow light and dark mode (my call).

Concise and quiet: no sidebar ads, banners or comment threads; one column of content; light and dark through
`prefers-color-scheme` (my call).

### Evidence

Every behaviour claim, relationship edge and "fires" entry carries a citation, written in MDX as a component:

- `<Ev scenario="add_filter_returns_true"/>`: a compat scenario, in `compat/<area>/scenarios.json` at the pinned
  server commit;
- `<Ev break="wp_filter_replace_wholesale"/>`: a documented break, in `compat/<area>/breaks.json`;
- `<Ev test="speedyspec-wp-hooks-php::wordpress::…"/>`: a SpeedySpec test;
- `<Ev source="wp:wp-includes/plugin.php#L120-L134"/>`: lines of WordPress 7.1.5's own code, from the release
  tarball;
- `<Ev source="ss:packages/wp-plugin/wp-plugin-legacy/functions/plugin.php#L40-L52"/>`: lines of SpeedySpec's code
  (the `packages/` tree or speedyspec-server) at the pinned commit.

WordPress pages cite only `wp:` sources. SpeedySpec pages may cite both.

**Tiers** apply to SpeedySpec pages (my call):

- **Ported seams: verified.** Every claim on a SpeedySpec page cites a scenario, break or test. A source citation
  alone fails the build.
- **Unported seams: read from source.** Claims may cite code lines. The page says so in its provenance line. A claim
  is upgraded to verified when a scenario covers it.

WordPress pages describe WordPress's own code, so they may cite `wp:` sources in any seam. Their provenance line says
*verified against WordPress 7.1.5* for claims backed by a recorded scenario, and *read from WordPress 7.1.5's source*
otherwise.

**Verdicts for unported symbols** (my call). An unported symbol's verdict is computed, not claimed:

- **Same code as WordPress** when its definition in the `packages/` tree parses to the same syntax tree as in 7.1.5
  (ignoring comments and whitespace). The citation is that comparison, done at build time. Its examples then run on
  WordPress only, because the runtime runs the same code.
- **Modified in SpeedySpec** when the trees differ. The page shows the diff, and the symbol needs a written
  difference with a citation before its verdict can become *works as WordPress* or *works with differences*.
- **Removed in SpeedySpec** when it is in 7.1.5 but not in the `packages/` tree or the runtime. This also needs a
  written reason.

Unported symbols get no `/speedyspec/` page of their own, only a row in their seam's compatibility table, until they
are modified, ported or removed (my call: no thousands of near-empty pages).

The build checks every citation exists at the pinned versions, and that each source citation's lines still hold the
text they held when cited (a stored hash of the cited lines), so a moved line fails instead of pointing at the wrong
code (my call).

**Coverage.** Every symbol in the inventory must be in a seam (above). A symbol without a written page still gets a
page with its signature, seam, role and history, marked *not yet documented*. The build writes a coverage report per
seam. A **ported** seam must be at 100% or the build fails. Unported seams may be partial (my call: full WordPress
coverage is the goal, but thousands of pages cannot block every build until all are written).

### Examples

- Each example is a PHP file in `seams/<seam>/examples/<symbol>/<name>.php`, with a sibling `<name>.expect.json`
  holding the expected output and return value. The page embeds it by name, so the code shown is the code run.
- Each example declares where it runs: `wordpress` (WordPress 7.1.5), `runtime` (SpeedySpec), or both. A page whose
  verdict is *works as WordPress* runs its examples on both. A porting example's "before" runs on WordPress only,
  its "after" on both.
- **The runner is the guide's own Rust crate, `speedyspec-guide-examples`** (Jacob, 2026-11-20), in the guide repo
  under `runner/`. It depends on `speedyspec-php` and the hook-area crates from the pinned speedyspec-server checkout
  (path dependencies through a `runner/server` symlink), linked to the server's static PHP. It is a plain executor: it
  runs the PHP files of a manifest on one side, in a fresh request each, and prints what each printed, returned and
  threw. The TypeScript toolchain owns the example files, the manifest, the comparison and the reports.
  - WordPress side: WordPress's files for the seam (`seam.yaml` `wordpress`) are loaded from the release the guide
    already downloads and verifies, as the compat recorder loads them. (`speedyspec-compat`'s own download and loading
    code is private, so the runner doesn't depend on it; found while planning, 2026-11-20.)
  - Runtime side: the native areas the runtime worker boots (`speedyspec-php-runtime/src/worker.rs`). A test reads
    `worker.rs` at the pin and fails when the runtime boots an area the runner doesn't.
  - CLI: `speedyspec-guide-examples run --side <wordpress|runtime> --manifest <file>` and
    `reflect --mode <baseline|runtime>`. The `examples` stage writes `generated/examples.json`; the site reads it, so the
    "runs in CI" mark is that file, not a promise.
  - It never edits speedyspec-server. If that session later adds snippets to `speedyspec-compat`, the crate can become a
    thin wrapper over it or go away.
- CI runs every example on every pull request. A nightly job runs them against speedyspec-server's `main` and opens an
  issue when one fails, so breakage is seen before the pin moves (my call: an issue, not a failed build).

### History

- **Added in / removed in**, for SpeedySpec, are computed. Each run records the runtime's API (Reflection) at the pinned
  server; the run counts as a release only when the pin is the commit a `speedyspec-php-runtime-v*` tag points to, and
  otherwise records `unreleased`. The history is kept in a committed `generated/speedyspec-history.json`. (Walking and
  building every runtime tag was the first plan; recording per pin was chosen while planning, 2026-11-20, because no tag
  exists yet and building each tagged runtime is costly. A release the pin skips is dated to the next recorded one.)
- **WordPress's since** is computed the same way from WordPress's release tags, never from `@since`. The tags come
  from the upstream `WordPress/wordpress-develop` repository, fetched read-only, because `SpeedySpec/wordpress-develop`
  does not carry them (my call). The tool walks every release tag back to the first, parsing each with the same
  extractor. Results are kept in a committed `generated/wordpress-history.json`, so only tags not yet in it are
  parsed on later runs.
- **Changed in** and **deprecated in** are written by hand on the page, each citing the release, the commit or pull
  request, and the scenario or test that shows the new behaviour.
- The build fails if a page names a version that has no tag: SpeedySpec versions against `speedyspec-php-runtime-v*`,
  WordPress versions against WordPress's release tags.

### Inventory

The list of every symbol and its exact signature:

- **WordPress:** the PHP extractor parses the **WordPress 7.1.5 release**, the tarball pinned in `sources.json`, the
  same one the compat fixtures are recorded against (my call: WordPress pages describe WordPress as it is, and the
  `packages/` tree is SpeedySpec's modified fork, which no longer holds the hook functions or `WP_Hook`). It lists
  functions, classes, methods, constants, globals, and hooks (calls to `apply_filters*` and `do_action*`; dynamic names
  like `pre_option_{$option}` kept as patterns). It reads signatures only, never comments.
- **SpeedySpec's fork:** the same extractor parses the pinned `packages/` tree. Comparing it with the 7.1.5 inventory
  gives each unported symbol's *same / modified / removed* verdict (see "Evidence").
- **SpeedySpec:** what the runtime registers in PHP (native functions, classes, `#[php_port]` twins), listed with PHP
  Reflection run inside the runtime by the guide's runner crate, so the signatures are what PHP code actually
  sees (my call).
- Output: `generated/inventory.json`, not hand-edited, rebuilt on every build.

### Tooling architecture

Patterns, per the design principles:

- **Hexagonal architecture** for the tooling. The domain is plain TypeScript objects: `Symbol` (function, class,
  method, hook, global), `Seam`, `Role`, `Edge`, `Dependency` (with its `CallSite`), `Verdict`, `Citation`, `Example`, `HistoryEntry`, with `Version` and
  `Citation` as **Value Objects**. It depends on nothing.
- **Ports**, in their own package `@speedyspec-guide/contracts`: `SymbolSource`, `ReleaseHistory`, `EvidenceSource`,
  `ExampleRunner`.
- **Adapters**, each in its own package: `php-parser-source` (WordPress), `reflection-source` (SpeedySpec, through
  the runner crate), `git-tag-history`, `compat-files-evidence` (scenarios, breaks, fixtures), `runner-crate-examples` (calls `speedyspec-guide-examples`).
- **Pipes and Filters** for the build: extract → assign to seams → validate → generate → `astro build`. Each stage
  reads the last stage's output files, so any stage can be run and inspected alone.
- **Specification** for validation: each rule (`EverySymbolInOneSeam`, `CitationExists`, `CitedLinesUnchanged`,
  `PortedSeamVerified`, `PortedSeamCovered`, `VersionHasTag`, `ExampleHasRun`) is one object, combined into the build
  check, and reported together so one run shows every failure.
- **Repository** over the content collections (`SeamRepository`, `SymbolPageRepository`) so pages are read the same
  way by the validator and the site.

### Requests to speedyspec-server

The guide must not edit speedyspec-server. These go to that repository's session as requests:

1. **Optional, later: snippets in `speedyspec-compat`.** If that session wants it, the guide's runner can move into
   the harness. The guide does not wait for it.
2. **Stable library surface.** The runner uses `speedyspec-php`'s public items (`Php`, `BootOptions`, `NativeArea`) and
   the area constructors the worker uses. Ask that session to treat them as used by the guide, so a change to them
   comes with a note. The nightly run against `main` catches a break either way.
3. **Make the request-scoped hook registry public.** `RequestHooks` and `RequestHooksArea` (worker.rs) empty the
   `SpeedySpec\\Hooks\\Hook` registry at the end of each request. They are private to the runtime binary, so the runner
   carries a copy; without it, a callback from one example outlived its request and crashed the next (found in review,
   2026-11-20).

The SpeedySpec inventory (Reflection of what the runtime registers) also runs through the guide's runner: it is one
more PHP file run on the runtime side, which prints the registered functions and classes as JSON.

### Order

Reference before guides, because guides are mostly links into reference pages and the reference is where the
evidence rules are proven:

1. **Foundations:** the repository, the extractor and inventory, the seam map, the build checks, and the examples
   runner crate.
2. **Hooks reference:** every Hook API function, `WP_Hook` and each method, `$wp_filter`, with examples, on the
   WordPress and SpeedySpec sides. Hooks is ported and has compat data, so it proves the verified tier.
3. **Hooks guides:** porting guide and checklist, then the architecture page, linking into step 2.
4. **Every WordPress symbol** gets at least its generated page, in its seam. Then Options, as its port lands.

## Left to the builder

- Visual design within the approved layout: typography, colours, spacing.
- How Pagefind indexes WordPress names and aliases.
- Exact file formats of `seam.yaml`, `*.expect.json` and the generated files.
- The split of any `packages/wp-*` package into more than one seam, as each seam is written.
- CI provider details (GitHub Actions, as the other repositories), caching the PHP build and WordPress download.
