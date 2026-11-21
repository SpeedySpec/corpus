# Exploration: A block library with replaceable outputs and a replaceable editor

**Status:** built
**Created:** 2026-11-18
**Author:** Claude, for the project owner
**Context:** 47 shared block types in six areas, each with replaceable default outputs, plus an editor seam.
**Approved by:** not recorded
**Socialized with:** not socialized

---

## Problem Statement

The block system had seven blocks; the compared plugins ship 30 to 80 each, plus editing, styling and data features.
Site builders could arrange blocks but not write a page. The project owner wanted the library available but not baked
in. A plugin's "own take" on a block should replace the output but keep the settings, so pages still open. Each
plugin's blocks should be grouped in a searchable section. The editor should be replaceable, or absent.

## Current State

- `SpeedySpec/speedyspec-api-block` after PR #7 (merge `7f34fef478`): the [block model](2026-11-17-block-model.md),
  its phases 2 to 7, and a reference builder with drag and drop and undo.
- Any package could already register an output for any type. Nothing recorded who drew what.
- Block types declared asset names, but nothing mapped a name to a file.
- The builder API marked every block in admin renders, whether or not an editor was in use.
- `block-structure` held both the definitions and the outputs of the five structural blocks.

## Proposed Solution

Split the library by area. Each area has a definitions package (types, schemas, behaviours, migrations, no outputs)
and a default outputs package (outputs and their assets). A plugin's own take depends on the definitions only and
registers outputs; the last registration wins.

### Architecture

```
  block-text-domain ──defines──> core/heading, core/paragraph, ...
        ▲                              │
        │ depends on                   │ drawn by (last registered wins)
  block-text (defaults)  ─────────────>│
  plugin's own take      ─────────────>┘   outputs declare their own assets
                                           │
  host: AssetRegistry + AssetLocator ──> page assets      EditorInterface (zero or one bound)
                                                          └─> sets RenderRequest.editing for admin renders
```

Areas: structure, text, media, actions, interactive, site.

### Data Model

- **Formatted text:** a string setting with `x-control: rich-text`, holding a small HTML subset (`strong`, `em`, `s`,
  `code`, `sub`, `sup`, `mark`, `br`, and `a` with `href` only).
- **Media settings** store ids, not URLs: `x-control: media` with an `x-media-kind`.
- **Assets:** a name (`vendor/name`), a kind (script or style), a source (inline, URL, or a file in a package) and
  dependencies.
- **Lock:** the `core/lock` extension, `{move, remove}`.
- **Transforms:** declared on a block type, mapping each target setting to a source setting.

### Interfaces

- `DeclaresAssetsInterface::assets()`, on outputs.
- `BlockPackageLoaderInterface::outputOwner(BlockName, Mode, target)`.
- `AssetRegistryInterface::resolve(names)` and `AssetLocatorInterface::url(package, path)`.
- `RichTextSanitizerInterface::clean(html)` and `OutputEscaperInterface::richText(value)`.
- `EditorInterface::open(EditorRequest): EditorPage`.
- `RenderInput::editable(setting)`, which adds `data-block-setting` only while editing.
- `DocumentStepInterface::context(Document, Context, Mode)`, run once before any block resolves.
- `MediaLibraryInterface` (find, search, upload) and `SiteMapInterface` (address, breadcrumbs, search target).

## Design Decisions

1. **Split by area into definitions and default outputs.** Chosen by the project owner. A plugin can replace one
   block, one area or all, and a site loads only the areas it uses. *Trade-off:* more packages, and moving a block
   between areas is breaking, so each block's area was fixed up front.
2. **Assets belong to whoever draws.** Replacing an output stops its assets loading. *Trade-off:* library
   definitions declare no assets, unlike the example package.
3. **One editor per site through a host contract.** Recommended by the designer; the project owner did not object.
   It gives one way to turn the editor off or swap it. *Trade-off:* no per-document-kind editors.
4. **Editing marks only while an editor is bound.** A visitor's page never carries block ids or setting marks.
   *Trade-off:* a site with no editor gets admin renders without marks.
5. **Formatted text as an HTML subset.** The designer's call, seen by the project owner. Every editor can produce it
   and every output print it. *Trade-off:* a sanitizer on both edges, when edited and when drawn.
6. **"Not set up" is not broken.** A block with nothing chosen yet draws nothing for visitors and a prompt for
   editors, with no issue. The package suite needs examples to resolve cleanly, and definitions cannot know a host's
   media ids. *Trade-off:* a missing setup is not reported as an issue.
7. **No script needed for content.** Accordions use `details`, the modal uses `popover`, and tabs show every panel.
   *Trade-off:* the modal does not make the rest of the page inert.
8. **Uploads judged by content.** Type comes from the first bytes; SVG and HTML are refused. *Trade-off:* only a fixed
   list of formats is accepted.
9. **Transforms as data, locks enforced by the server as an extension.** Every builder gets the same result and the
   document format is unchanged. *Trade-off:* `restore` skips lock checks so undo works, so a client can bypass a
   lock.
10. **Social links drawn by name, not logo.** Logos are trademarks. *Trade-off:* a plugin wanting logos ships its own
    outputs package.

## Implementation Phases

1. **Foundations.** Package titles and sections, output ownership, output-declared assets, the asset registry, rich
   text, editing marks, the editor seam, document steps, the suite's definitions and outputs modes, and the
   `block-structure` split. *Delivers:* the seams every area uses. Shipped (PRs #8 and #9).
2. **Text.** Nine text blocks and typing on the page with a formatting bar. *Delivers:* writable pages. Shipped
   (PR #10).
3. **Media.** The media contract with memory, filesystem and WordPress libraries, upload checks, media routes, nine
   blocks, and a picker. *Delivers:* images, video, audio, embeds, icons and maps. Shipped (PR #11).
4. **Actions.** Buttons, call to action, testimonial, pricing, rating and social links, with a stylesheet shown on
   the editor's canvas. *Delivers:* marketing blocks. Shipped (PR #12).
5. **Interactive.** Accordion, tabs, slider, countdown, progress, counter and modal, with an optional script and a
   visitor page in the demo. *Delivers:* interactive blocks that work without script. Shipped (PR #13).
6. **Site.** Navigation, table of contents, breadcrumbs and search, the site map contract, shared heading anchors and
   the post-list template. *Delivers:* site-level blocks. Shipped (PR #14).
7. **Editor features.** Copy and paste, templates, binding from the inspector, stored tokens and breakpoints with a
   site styles screen, multi-select, transforms and locks. *Delivers:* a complete reference editor. Shipped (PR #15),
   with follow-up fixes (PR #16).

## Fleet Impact

- `SpeedySpec/speedyspec-api-block`: adds twelve area packages (six `-domain`, six outputs) and changes `block-domain`,
  `block-app`, `block-presentation`, `block-builder`, `block-contract-tests`, the infra packages and
  `block-infra-wp`. Confirmed from merged PRs #8 to #16 and the package folders present.
- `block-builder` became a mixed package with a PHP `ReferenceEditor`.
- WordPress: `block-infra-wp` gained media and site map implementations only; no admin screen or editor wiring.
- Other SpeedySpec repositories: unverified; the sources name none.

## Open Questions

1. **Can a lock be made safe against direct `restore` calls?** A separate lock check that undo can pass would settle
   it.
2. **Should `Asset::file` be confined to its own package's files?** A vendor check at registration would settle it.
3. **Should `outputOwner()` track outputs set straight on the registry?** Recording ownership in the registry would
   settle it.
4. **Should the binding escape be per target?** It is HTML-specific and runs at resolve time.
5. **How should relative links and non-Latin heading anchors behave?** `page.html` becomes `https://page.html`, and
   non-Latin headings all get `heading`.
6. **Should headings inside a repeat be unique against the page?** They are not yet.
7. **How should hosts drawing several documents with the same block ids avoid clashing element ids?** Not addressed.
8. **Should the demo's upload route check size before reading and check request forgery?** Deferred.
9. **Should countdown text be translated and pluralised?** It is English only.
10. **Which deferred features come next?** Candidates: brand logos, more embed providers, image resizing, a mobile
    menu button, transforms that change children, binding inside lists, and site styles in undo.

## Success Criteria

- [x] A page made with one output package opens with another — definitions are the contract; pinned by outputs-mode
  suite tests.
- [x] A replacement output wins and its assets replace the default's — pinned in part 1.
- [x] Formatted text keeps only the allowed subset, and bound values stay text — pinned in part 1.
- [x] Editing marks never reach visitors — pinned in part 1.
- [x] Uploads that lie about their type are refused — pinned in part 3.
- [x] Interactive blocks work with no script — checked by hand on the demo's visitor page in part 5.
- [x] Heading anchors are unique on a page — pinned in part 6.
- [x] Locked blocks cannot be moved or removed, while undo still works — pinned in part 7.
- [ ] The editor checked in Safari and Firefox — the Shift-click fix was only tested in one browser.

---

## Non-Goals

- **Forms** — a separate project built on this one.
- **A WordPress-compatible block library** — names and settings are this system's own.
- **Pixel-matching any plugin** — defaults are plain, accessible markup.
- **WordPress admin wiring** — the project owner chose the demo host only.

## Alternatives Considered

### Option: Definitions and outputs in two packages for the whole library
- **Pros:** simple.
- **Cons:** a plugin cannot leave out one area's defaults.
- **Verdict:** rejected.

### Option: One package with both, plugins override outputs
- **Pros:** fewest packages.
- **Cons:** defaults always load, with their styles and assets.
- **Verdict:** rejected.

### Option: Let each host load any editor script
- **Pros:** no new contract.
- **Cons:** no uniform way to turn the editor off or swap it.
- **Verdict:** rejected for `EditorInterface`.

### Option: A structured text format
- **Pros:** no HTML to clean.
- **Cons:** harder for in-page editors to produce and outputs to print.
- **Verdict:** rejected for a cleaned HTML subset.

## Dependencies

- The block model with phases 2 to 7: style vocabulary, `core/repeat`, document references, the builder API and the
  reference builder.
- The record store from the storage addendum, for stored tokens and breakpoints.

## References

- [Block model exploration](2026-11-17-block-model.md)
- [Block plugin seams exploration](2026-11-17-block-plugin-seams.md)
- [SpeedySpec/speedyspec-api-block](https://github.com/SpeedySpec/speedyspec-api-block)
- PRs: [#8](https://github.com/SpeedySpec/speedyspec-api-block/pull/8),
  [#9](https://github.com/SpeedySpec/speedyspec-api-block/pull/9),
  [#10](https://github.com/SpeedySpec/speedyspec-api-block/pull/10),
  [#11](https://github.com/SpeedySpec/speedyspec-api-block/pull/11),
  [#12](https://github.com/SpeedySpec/speedyspec-api-block/pull/12),
  [#13](https://github.com/SpeedySpec/speedyspec-api-block/pull/13),
  [#14](https://github.com/SpeedySpec/speedyspec-api-block/pull/14),
  [#15](https://github.com/SpeedySpec/speedyspec-api-block/pull/15),
  [#16](https://github.com/SpeedySpec/speedyspec-api-block/pull/16)
- Source spec: "2026-11-18-block-library-design.md"
- Source plans: "2026-11-18-block-library-part-1.md" to "2026-11-18-block-library-part-7.md"
