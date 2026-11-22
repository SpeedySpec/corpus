# ADR-027: The guide is written from behaviour and code, never from docblocks

**Status:** Accepted
**Date:** 2026-11-21
**Deciders:** both: the project owner (PHPDoc is never a source; Rust docs are kept by hand; no doc attributes in
WordPress's PHP); the designer (the reasons for Rust doc comments, and the enforcement)
**Exploration:** [The SpeedySpec Guide](../../../explorations/2026-11-20-speedyspec-guide.md)
**Related:** [ADR-017](017-php-twins-and-the-override-flag.md), [ADR-021](021-recorded-compatibility.md)

---

## Context

The SpeedySpec Guide documents every WordPress symbol and every symbol SpeedySpec adds, and says for each one whether
it still works on SpeedySpec. Both codebases already carry documentation in comments: WordPress has PHPDoc on almost
every function, class and hook call, and the Rust crates have rustdoc comments. Generating the guide from those
comments looks like the cheap way to cover more than 16,000 symbols and keep up with change.

Three ways to do that came up:

1. Render WordPress's PHPDoc, and the hook docblocks just above each `apply_filters()` and `do_action()` call, as the
   reference text, the way developer.wordpress.org does.
2. Replace WordPress's docblock tags (`@since`, `@param`, `@return` and the rest of the PHP-FIG PHPDoc tags WordPress
   uses) with PHP attributes, add WordPress-specific attributes for filters and actions, and read those attributes.
3. Generate the Rust side's pages from rustdoc, with a `#[wp_port]` attribute or a docblock manifest linking each
   Rust item to the WordPress symbol it ports.

### Why this matters

The guide promises that every claim is backed by evidence or a cited line of code. If any page restates a comment,
that promise breaks quietly: a wrong `@since`, a stale `@return` or a hook docblock listing the wrong arguments would
be published as fact. Porters would trust it, because nothing on the page would show it came from a comment.

---

## Decision

The guide reads code and recorded behaviour, never comments. Reference text is written by hand from verified behaviour
or from reading the code, and versions are computed from release tags. Neither PHPDoc nor rustdoc is a source, and
no attributes are added to WordPress's PHP to stand in for them.

### Rules

1. **PHPDoc is never a source.** A page states what a recorded scenario, an example run in CI, a WordPress test or a
   cited line of code shows. A docblock can be wrong, and it does not count as a citation.
2. **Since comes from tags.** WordPress's *added in* is computed from WordPress's release tags, and SpeedySpec's from
   the `speedyspec-php-runtime-v*` tags. `@since` is never read. *Changed in* and *deprecated in* are written by hand,
   each citing a release and the scenario or test that shows the change.
3. **The inventory reads code only.** The extractor lists signatures and hook calls, never comments. A symbol's hash
   is taken from its syntax tree with comments stripped, so a comment edit never changes a verdict.
4. **No doc attributes in WordPress's PHP.** WordPress's docblocks are not converted to PHP attributes, and no
   attributes are added for hooks.
5. **Rustdoc stays with the Rust code.** It documents Rust items for people using the crates. The guide documents what
   PHP code sees: the runtime's PHP API, listed by PHP Reflection inside the runtime, and its behaviour, shown by
   examples and recorded scenarios. Architecture pages are written by hand, and only the crate and package graph is
   generated.
6. **The Rust docs are written by hand.** No `#[wp_port]` macro and no docblock manifest generates them.

### What this looks like

✅ **Correct**

```mdx
---
symbol: "function:add_option"
---

Stores a new option and returns `true`. An option that is already stored is left as it is, and it returns `false`.
<Ev scenario="add_option_existing" area="options"/>
```

The claim is backed by a recorded scenario, and the page has no *since* typed by hand.

❌ **Not permitted**

```mdx
Adds a new option. Since 1.0.0.

@param string $option Name of the option to add.
@return bool True if the option was added, false otherwise.
```

This is WordPress's docblock restated as the page: no evidence, and a version the build did not compute.

### Patterns

- **Single source of truth:** behaviour comes from recordings and examples, versions from tags, signatures from the
  syntax tree. A comment is a second copy that can drift, so it is not used.
- **Golden Master** ([ADR-021](021-recorded-compatibility.md)): recorded scenarios back the behaviour claims.
- **Ports and Adapters:** each input enters the guide's toolchain through its own adapter (the PHP parser, git tag
  history, the examples runner, PHP Reflection), and no adapter for comments exists.

---

## Consequences

### Positive

1. Every published claim can be traced to evidence or a line of code, and the build checks the citations.
2. A wrong or outdated docblock in WordPress cannot leak into the guide.
3. WordPress's source stays as upstream wrote it, so merging upstream trunk does not conflict on docblocks.

### Negative

1. Writing 16,000+ pages by hand is slow.
   **Mitigation:** every symbol still gets a generated page with its signature, seam, role and history, marked *not
   yet documented*. Only ported seams must reach 100%, and the coverage report shows the gaps.
2. Hand-written Rust docs and guide pages can fall behind the code during the first pass.
   **Mitigation:** the project owner decided the drift is a one-time problem while the docs are first written. Once
   written, they change too rarely to need a generator. Docs are updated in the same change as the code.
3. Hook argument descriptions that WordPress's docblocks already hold are written again.
   **Mitigation:** the inventory lists every hook call with its arguments from the code, so only the meaning is
   written by hand.

### Trade-offs

**We prioritise:**
- Claims that are true and checked.
- An unmodified WordPress source.

**We accept:**
- Writing the reference text by hand.
- Repeating some of what good docblocks already say.

---

## Fleet impact

- **speedyspec-guide:** the extractor (`adapter-php-parser-source`) reads code only. The history stages compute
  versions from tags. Pages in `content/` are hand-written MDX keyed by `symbol:`.
- **wordpress-develop:** no doc attributes and no docblock rewrites. Docblocks are left as upstream wrote them.
- **speedyspec-server and speedyspec-cli:** rustdoc is written and updated by hand alongside the code, with no
  generator. The guide reads the runtime through Reflection, not rustdoc.

---

## Enforcement

### Automated

- The extractor, `adapter-php-parser-source/extractor/extract.php`, reads code only, so no comment text reaches
  `inventory-wp.json`.
- The `validate` stage's citation rules fail a citation that does not exist at the pinned versions, and one whose
  cited lines changed since it was cited (stored hash).
- `VersionHasTag` fails any WordPress or SpeedySpec version named on a page that has no release tag.
- No tool detects prose copied from a docblock. Review checks that.

### Review checklist

- [ ] Every claim on the page cites a scenario, an example, a test or a line of code, never a docblock.
- [ ] No *since* or *added in* is typed by hand. *Changed in* and *deprecated in* each cite a release and evidence.
- [ ] The change adds no doc attributes to WordPress's PHP and does not rewrite its docblocks.
- [ ] Rust doc comments changed with the Rust code they describe.

---

## Exceptions

An API WordPress bundles but another project documents, such as sodium_compat, gets no pages. Its seam links to that
project's documentation instead (`elsewhere` in `seam.yaml`). The project owner decides each one.

---

## Revision history

- **2026-11-20:** PHPDoc ruled out as a source, and *since* computed from tags, in the guide design.
- **2026-11-21:** PHP doc attributes and generated Rust docs ruled out. PHP attributes can only go on declarations,
  never on a call such as `apply_filters()`, and converting WordPress's docblocks would conflict with every upstream
  merge. The project owner chose to keep the Rust docs by hand. Recorded as an ADR.
