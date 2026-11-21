# Exploration: What popular block plugins rebuild around WordPress

**Status:** complete
**Created:** 2026-11-17
**Author:** Claude, for the project owner
**Context:** A source-reading comparison of 14 free WordPress block plugins, to find the seams a neutral block system needs.
**Approved by:** not recorded
**Socialized with:** not socialized

---

## Problem Statement

The project wanted a block system that does not depend on WordPress and that any block plugin could build on. Nobody
knew which parts of WordPress's block system plugins actually lean on, which parts they rebuild, and where their
designs truly differ. Without that, a new model risked copying WordPress's quirks or missing a seam every plugin
needs. The answer was for the designers of the block model, written the same day.

## Current State

- WordPress renders blocks through `render_callback` functions and lets plugins change output with the `render_block`
  and `render_block_data` filters. Parent blocks pass values down with `providesContext` and `usesContext`.
- WordPress offers a block bindings API and an interactivity API, but the compared plugins barely use them.
- Each plugin is a builder experience fused to its own blocks. A block written for one cannot be used in another.
- In `wordpress-develop`, the project's block code was the `wp-block-legacy` package: WordPress's classes, with no
  contracts.

## Proposed Solution

The report recommends ten properties for a neutral block system:

1. **Structured settings.** Settings read out of saved HTML are a 1.4% edge case and belong in an adapter.
2. **System-assigned block ids,** unique within a document and regenerated on copy or import.
3. **Namespaced context keys and a typed current item,** so loops stop using global state.
4. **A registry of data sources** with typed arguments, a fallback per binding, and one escaping step per target.
5. **A declared seam for features on many block types,** replacing registration patching.
6. **Ordered rendering stages others can join:** migrate settings, evaluate conditions, resolve bindings, produce
   output, transform output, collect assets. This replaces the `render_block` filter.
7. **Shared presentation vocabulary, not shared implementation:** one responsive value shape with a breakpoint
   registry, design tokens, and a per-type asset list. CSS generation stays with renderers.
8. **Documents loadable by id from trusted storage,** able to reference other documents with a loop guard.
9. **Settings that never leave the server,** such as a notification recipient.
10. **For forms:** stable field keys independent of labels and order; the definition from storage, never from the
    request; a validator per field type; a spam-check contract; an ordered pipeline of actions; and no visitor input
    in recipients or headers.

### Architecture

```
stored document ──> prepare/migrate ──> gate (conditions) ──> resolve bindings ──> render ──> transform ──> collect assets
                         │                    │                     │                │
                  defaults, versions   condition registry    data source registry   renderer per target
```

### Interfaces

The research notes sketch the shapes, not final signatures:

- a data source: an id, an argument schema, a return type, and `resolve(args, context)` returning a value or nothing;
- a binding on a block: `{source, args, fallback}` per setting;
- a condition: a type and `evaluate(args, context)`, composed as groups of rules, with no expression strings;
- a renderer per target: `render(node, resolvedSettings, renderedChildren, context)`.

## Design Decisions

1. **Use the 14 plugins of one published roundup as the sample.** It gave a broad, outside-chosen spread from 2 to
   84 block types. *Trade-off:* the sample reflects one article's picks, not usage data.
2. **Read free editions only, as source, without installing or running them.** This is fast, repeatable and safe.
   *Trade-off:* paid features and editor-side behaviour had to be inferred, and form weaknesses are unconfirmed.
3. **Count each seam across all 14 plugins.** Counts separate what every plugin needs from one plugin's habit.
   *Trade-off:* a count says nothing about how well a plugin does it.
4. **Split the work into four areas, one note each:** rendering and dynamic data, extension and layout, forms, and
   styling and assets. *Trade-off:* some seams, such as visibility, appear in several notes.

## Fleet Impact

- `SpeedySpec/speedyspec-api-block`: the block model design cites this report as its research, and its contracts
  follow the recommendations (known from the design's own text).
- The `wp-block-legacy` package in `wordpress-develop` is unchanged by the research itself.
- Any future forms project: unverified, since forms were not designed in these sources.

## Open Questions

1. **How do the plugins generate ids and responsive attributes in the editor?** Reading unminified editor sources
   would settle it; only one plugin's id scheme was confirmed from a bundle.
2. **What do the paid editions add for dynamic content and conditions?** Inspecting the paid plugins would settle it.
3. **How does Vayu deliver its generated CSS?** Its hooks were commented out in the inspected copy; running it would
   settle it.
4. **Are the form weaknesses real?** Running each form system against crafted submissions would settle it.

## Spike Findings

**Settles:** how common each seam is across the 14 plugins.
**Built:** a read-only scan over the unpacked plugins counting `block.json` keys, hook calls and attribute sources.
**Established:**
- 446 `block.json` files across all 14 plugins;
- 5,092 of about 5,200 attributes stored in the block's comment data, not parsed from HTML;
- `render_block` used by 11 plugins and `render_block_data` by 7, with `pre_render_block` used by none;
- `parent` used by 11 plugins, `providesContext` by 10, and `usesContext` by 9;
- WordPress's tag processor used by 8 plugins;
- `register_block_bindings_source` used by none.

**Does not establish:** how each feature behaves at runtime, or anything in the paid editions.
**Consequence:** structured settings, system-owned stages and a data source registry became the design's base.

## Success Criteria

- [x] Every listed plugin downloaded and read — the report lists all 14 with versions.
- [x] Each seam counted across the sample — the report's table gives a count per seam.
- [x] Real differences named, not only common ground — the report has a "where they differ" section.
- [x] Recommendations a design can use — the block model design cites the report and follows it.
- [ ] Paid editions and editor code inspected — stated as a limit of the research.

---

## Non-Goals

- **Ranking the plugins** — the goal was seams, not a buyer's guide.
- **Paid add-ons** — not in the free downloads.
- **Running the plugins** — the comparison was by reading only.

## Alternatives Considered

The research compared the plugins' approaches rather than options for this project.

### Option: Tokens in content
- **Pros:** survives static HTML; used by GenerateBlocks, Otter and Greenshift.
- **Cons:** five syntaxes in five plugins; a security surface that needs escaping per context.
- **Verdict:** keep only as an import adapter.

### Option: Structured bindings with a source registry
- **Pros:** typed, extensible, escaped once per target; GenerateBlocks' registry is the cleanest example.
- **Cons:** each bindable setting needs schema support.
- **Verdict:** recommended.

### Option: Global post state for loops
- **Pros:** what WordPress offers; used by 8 plugins.
- **Cons:** ties loops to posts and to WordPress.
- **Verdict:** replace with an explicit current item.

### Option: Rewriting rendered HTML as a string
- **Pros:** works on any block; used by 11 plugins through `render_block`.
- **Cons:** fragile regular expressions and bypassed escaping.
- **Verdict:** replace with structured stages and an attribute bag.

### Option: Form definitions from the request
- **Pros:** simple; several plugins do it.
- **Cons:** lets a visitor shape what the server trusts; renamed labels change keys.
- **Verdict:** take definitions from trusted storage, as Kadence's advanced form does.

## Dependencies

- Free editions of the 14 plugins from the WordPress.org plugin directory.
- The published roundup that named the sample.

## References

- [Roundup of free WordPress block plugins](https://zoloblocks.com/best-free-wordpress-block-plugins/)
- [WordPress.org plugin directory](https://wordpress.org/plugins/)
- [Block model exploration](2026-11-17-block-model.md)
- Source report: "Block plugin seam comparison.md"
- Source notes: "dynamic-data-and-rendering.md", "extension-and-layout.md", "form-blocks.md", "styling-and-assets.md"
