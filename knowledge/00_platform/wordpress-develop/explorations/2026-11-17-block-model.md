# Exploration: A block model that does not depend on WordPress

**Status:** built
**Created:** 2026-11-17
**Author:** Claude, for the project owner
**Context:** The contracts that let any block work in any builder, with output supplied by the plugin that adds a block.
**Approved by:** the project owner on 2026-11-17
**Socialized with:** not socialized

---

## Problem Statement

The project owner wanted forms, themes, pages and posts built from blocks, with "a contract system so that all
plugins can use the same blocks system". Plugin authors had no shared contract: each block plugin rebuilt ids,
dynamic values, conditions, CSS, assets and forms on its own. The builder had to be separate from the blocks. Each
block's HTML had to come from the plugin that adds it. The admin had to draw blocks differently from the front end so
editors can see extra options. A later forms project also needed stable field identity and settings visitors never
see.

## Current State

- `wordpress-develop` held `wp-block-legacy`: WordPress's block classes, with no contracts.
- [The seam research](2026-11-17-block-plugin-seams.md) showed every compared plugin fusing a builder to its own
  blocks and working around WordPress's classes incompatibly.
- `SpeedySpec/speedyspec-api-block` held only an initial commit with a one-line README.

## Proposed Solution

Three roles. A **block package** carries block types, schemas, behaviour and output, and depends only on the domain.
A **builder** uses only builder contracts. A **host** picks packages and a builder, stores documents and supplies
data. A document is resolved through fixed stages, then drawn in `front` or `admin` mode.

### Architecture

```
            host (storage, data sources, chosen packages and builder)
              │
  block packages ──register──> registries (types, behaviours, outputs, decorators, sources, steps)
              │                         │
  document ──> Prepare ─> Gate ─> Resolve ─> Validate ─> View ─> Render ─> Collect
              (migrate,   (hide)  (bindings) (schema)   (behaviour) (output  (assets)
               defaults)                                            per type)
              │
  builder ──> catalogue, placement, editing operations, preview, templates
```

Each stage has a fixed core step and a typed extension point. Resolving runs stages 1 to 5; rendering adds 6 and 7.

### Data Model

- **Block type:** a `vendor/name`, an integer version, a JSON Schema for settings (with defaults and `x-private`), an
  optional view schema, children and parent policies, provided and used context keys, asset names, examples and
  catalogue metadata. It can be built from a plain array.
- **Block:** a system-assigned id, type and version, settings, extensions per decorator namespace, bindings
  (setting path to source, arguments and fallback), and ordered children. Blocks and trees are immutable.
- **Document:** a tree with an id, a kind (`page`, `post`, `template`, `part`, or a host kind) and a format version.
- **View:** plain values with no markup, plus admin context in admin mode.

### Interfaces

- `BlockPackageInterface::register(BlockPackageRegistrarInterface)`: a package's single entry point.
- `BlockOutputInterface::render(RenderInput): string`: one view drawn for one target and mode.
- `DocumentResolverInterface` and `DocumentRendererInterface::render(Document, RenderRequest)`.
- `DocumentEditorInterface`: insert, move, remove, duplicate, update settings or an extension, set or clear a
  binding, insert a template. Each returns a new document or a refusal.
- Builder contracts: `BlockCatalogueInterface`, `PlacementRulesInterface`, `BlockPreviewInterface`,
  `BlockTemplateRepositoryInterface`.

## Design Decisions

1. **A neutral data model, with blocks and builders apart.** Chosen by the project owner. The research found 98.6%
   of compared settings were already structured data. *Trade-off:* WordPress content needs an adapter.
2. **Output ships with the block, bound per target, mode and type.** A theme can replace one block's output without
   touching the block. *Trade-off:* any package may replace any output, so load order matters.
3. **Portability enforced by a shared test suite.** Composer cannot forbid a dependency, so the suite reads
   `composer.json` and fails on any runtime requirement besides PHP and the domain. *Trade-off:* a type added
   straight to the registry is outside the rule.
4. **Private settings.** They are absent from front-mode views and exports, present in admin mode, and never bindable.
   *Trade-off:* render and collect steps still see them, as trusted code.
5. **Gate runs before Resolve.** A hidden block's data sources are never called. *Trade-off:* gating settings cannot be
   bound.
6. **A bound value falls back.** The source value, the fallback, the stored value, then the default; the first that
   fits the schema wins. *Trade-off:* rejected values only show up as issues.
7. **A subset schema validator that refuses unknown keywords.** No rule is silently ignored. *Trade-off:* `oneOf`,
   `format` and similar are unavailable until a full validator is added.
8. **Decorators and an attribute bag instead of string rewriting.** Output is never edited as a string. *Trade-off:*
   each decorator needs a render step and each output must apply the bag.
9. **Resolving never throws for bad content,** and one resolve is capped at 50,000 blocks. *Trade-off:* nested repeats
   can spend the budget and stop the whole page.

## Implementation Phases

1. **Part 1a, model and resolving.** Types, documents, ids, decorators, migrations, bindings, behaviours and stages
   1 to 5, JSON serialization, client export, in-memory storage. *Delivers:* a resolvable document. Shipped (PR #1).
2. **Part 1b, output and the package contract.** Stages 6 and 7, the HTML escaper, the attribute bag, output
   registry, package loader, package suite, and an example package. *Delivers:* portable block packages. Shipped
   (PR #2).
3. **Part 1c, builder contracts.** Positions, one placement contract shared with the resolver, catalogue, templates,
   validated editing, preview, and an admin placeholder for broken blocks. *Delivers:* builders that cannot save a
   broken document. Shipped (PR #3).
4. **Phases 2 to 7 and storage.** A record store over memory, filesystem and PDO; an HTTP builder API; breakpoints,
   tokens and a style collector; conditions and `core/repeat`; structural blocks; `core/include`; and an experimental
   WordPress adapter. *Delivers:* a usable system around the core. Shipped (PR #4).
5. **Reference builder.** A framework-free JavaScript builder, then a redesign with editing on the page, drag and
   drop, and undo through a new `restore` operation. *Delivers:* proof a builder needs only the contracts. Shipped
   (PRs #5 to #7).

## Fleet Impact

- `SpeedySpec/speedyspec-api-block`: created by this work. It holds `block-domain`, `block-app`,
  `block-infra-memory`, `block-contract-tests`, `block-example`, and later storage, presentation, structure,
  WordPress and builder packages. Confirmed from merged PRs #1 to #7.
- `wordpress-develop`: the repository is cloned into its `packages/` folder; `wp-block-legacy` is not changed.
- `SpeedySpec/speedyspec-api-seo`: its service container is duplicated, not shared, and its brace style differs from
  the rules file. Known from the design's notes.

## Open Questions

1. **Should the service container be one shared package?** Extracting it touches SEO, hooks and options.
2. **How strict should style values be?** Legacy script vectors and `url(...)` in the attribute bag were left for the
   presentation phase.
3. **Should steps be kept from private settings, as outputs are?** Narrowing what steps receive would settle it.
4. **Serialization limits:** objects keyed `0..n-1` reload as lists, integers past 64 bits become floats, unknown keys
   are dropped. Choosing a richer format would settle it.
5. **Should a front export hide what visibility rules hide?** It cannot without the page's context; currently it does
   not.
6. **How should storage handle concurrent editors and revisions?** Neither locking nor listing documents exists, and
   undo restores whole documents.
7. **Can the WordPress adapter leave experimental status?** It needs its ports to carry inner blocks and context, and
   a run against a live WordPress.
8. **Who handles authentication, request forgery and rate limits for the builder API?** Left to hosts for now.

## Spike Findings

The reference builder's first interface was a probe of whether a structure-and-inspector builder was usable.
**Settles:** which editing model the reference builder uses.
**Built:** a three-pane builder (structure, inspector, sandboxed preview), checked by hand against a demo host.
**Established:** it was rejected as hard to work with.
**Does not establish:** whether third-party builders would find the contracts sufficient.
**Consequence:** the builder was redone with the page as the editor, using `data-block-id` marks in admin mode only.

## Success Criteria

- [x] No package needs WordPress — only `block-infra-wp` touches it, through ports.
- [x] A private setting never reaches a visitor — pinned by part 1a and 1b tests and the package suite.
- [x] An unknown block type round-trips unchanged and is stripped from front exports — pinned in part 1a.
- [x] Markup in a setting or bound value is escaped — pinned in part 1b and by the package suite.
- [x] A broken edit is refused with reasons and the document unchanged — pinned in part 1c.
- [x] Resolver and editor agree on placement — one placement contract, pinned in part 1c.
- [ ] The WordPress adapter is checked against a running WordPress — judged by reading only.

---

## Non-Goals

- **A full builder UI in this design** — builders are separate; the reference builder came later as proof.
- **Forms** — a separate project built on phase 1.
- **Reproducing WordPress's block library, supports, variations or patterns** — not needed for portability.
- **CSS generation in the core** — styling belongs to outputs until the presentation phase.
- **Client-side behaviour and hydration** — out of scope.

## Alternatives Considered

### Option: Mirror WordPress's model
- **Pros:** existing content and plugins map directly.
- **Cons:** WordPress's quirks become everyone's contract.
- **Verdict:** rejected.

### Option: Typed component classes
- **Pros:** strong typing per block.
- **Cons:** a block type could not be defined by data alone.
- **Verdict:** rejected.

### Option: A repository per storage engine
- **Pros:** each engine tuned on its own.
- **Cons:** every engine re-implements serialization and repositories.
- **Verdict:** rejected for one record-store contract with repositories written once.

## Dependencies

- The seam research of 2026-11-17.
- PHP 8.4, Composer path repositories and Pest, following the SEO monorepo's layout.

## References

- [Block plugin seams exploration](2026-11-17-block-plugin-seams.md)
- [Block library exploration](2026-11-18-block-library.md)
- [SpeedySpec/speedyspec-api-block](https://github.com/SpeedySpec/speedyspec-api-block)
- [PR #1, part 1a](https://github.com/SpeedySpec/speedyspec-api-block/pull/1),
  [PR #2, part 1b](https://github.com/SpeedySpec/speedyspec-api-block/pull/2),
  [PR #3, part 1c](https://github.com/SpeedySpec/speedyspec-api-block/pull/3),
  [PR #4, storage and phases 2 to 7](https://github.com/SpeedySpec/speedyspec-api-block/pull/4),
  [PR #5, builder redesign](https://github.com/SpeedySpec/speedyspec-api-block/pull/5),
  [PR #6, drag and drop](https://github.com/SpeedySpec/speedyspec-api-block/pull/6),
  [PR #7, undo and redo](https://github.com/SpeedySpec/speedyspec-api-block/pull/7)
- Source spec: "2026-11-17-block-model-design.md"
- Source plans: "2026-11-17-block-system-phase-1a.md", "2026-11-17-block-system-phase-1b.md",
  "2026-11-17-block-system-phase-1c.md"
