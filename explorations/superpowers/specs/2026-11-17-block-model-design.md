# Block system design

Date: 2026-11-17
Status: approved 2026-11-17; amended after the part 1a final review
Research: `.ai-research/reports/Block plugin seam comparison.md`

## Intent

Give the project a block system that does not depend on WordPress and that any block plugin can build on: a shared
set of contracts for saying that an area of a page is one kind of thing, configured by settings, filled with dynamic
information, drawn on the page by the plugin that added it, and arranged in whichever builder the site uses.

The project owner's words: forms should be built from blocks, "like Squarespace, like WordPress, like other UI
systems where areas of the page can be dictated to be one thing with dynamic information"; the system should allow
"building out customized themes, pages and posts"; "there must be a contract system so that all plugins can use the
same blocks system"; and the renderer, meaning "the UI that allows dropping and organizing blocks and building the
block templates", must be separate from the blocks, while "the rendering of the data into HTML should be part of
the plugin that adds the block." Finally, "the admin output needs to be different than the front-end output, so
that additional options context can be displayed when in the admin."

Two words are used precisely in this document. A **builder** is the UI for dropping, arranging and configuring
blocks and building block templates. **Output** is a block drawn as markup, in one of two **modes**: `front`, what
a visitor sees, and `admin`, what someone editing sees.

Today the block system here is `packages/wp-block/wp-block-legacy`: WordPress's classes with no contracts. A
comparison of 14 popular block plugins showed that each one is a builder experience fused to its own blocks, and that
each rebuilds the same things around WordPress's classes (block ids, dynamic values, conditions, CSS, asset loading,
forms), incompatibly. A block written for one cannot be used in another.

## Goals

- A block is portable across builders: a block package carries the block's definition, behaviour and output, and no
  builder UI, so the same block works in any builder.
- A builder is portable across blocks: the contracts tell it everything it needs to list, place, configure, preview
  and arrange a block it has never seen.
- A block type can be defined by data alone.
- Pages, posts, templates and fragments are documents: trees of blocks that can be built, saved, and loaded by id.
- Every change a builder makes goes through validated editing operations, so no builder can store a broken document.
- Any setting can be bound to a data source.
- A block is drawn differently in the admin than on the front end, and in the admin it can carry extra options and
  context for the person editing.
- A document can be resolved without being drawn, so other code (forms, at submission time) can read each block's
  final settings.
- Features that apply to many block types (conditions, animation, custom attributes) have a declared seam.
- Producing output is a sequence of typed stages others can join, replacing output filters.
- No WordPress function, class or package is needed by the core.
- Contracts and implementations are separate packages; an implementation is replaceable one contract at a time or as
  a whole layer.

## Non-goals

- Building a builder UI. This project delivers the contracts a builder uses; a builder (in the Astro admin or in a
  block plugin) is a separate project.
- Any form concept (field types, submissions). Forms are a separate project built on phase 1.
- Reproducing WordPress's block library, block supports, variations or patterns.
- Generating CSS in the core. Styling belongs to each block's output until the presentation vocabulary phase.
- Client-side behaviour and hydration.
- Wiring into a running WordPress site, or replacing `register_block_type()`.

## Anti-goals

- A block package that ships builder UI, or that only works in one builder.
- A builder that needs block-specific code to list, place or configure a block.
- Any step that rewrites a block's rendered output as a string. That is the post-processing this design replaces,
  and it bypasses escaping.
- Contracts shaped by WordPress's quirks: settings read out of saved HTML, comment-delimited markup as the canonical
  format, global state for the current loop item.
- Block types that can only be defined by writing a PHP class.
- Any path where a stored setting or bound value reaches output without being escaped for the target.
- Anything meant only for the admin (private settings, option lists, editing context) appearing in front-end
  output.
- Expression evaluation of stored strings (one compared plugin uses `eval()` for display logic).

## Constraints

- PHP 8.4, `declare(strict_types=1)`, 4-space indentation, the docblock rules in `.claude/rules/coding-style.md`.
- That rules file says every opening brace goes on its own line. The SEO packages put control-structure braces on
  the same line. These packages follow the rules file as written; bringing the SEO packages into line is separate.
- The domain package requires only PHP and has no `final` classes.
- Namespace root `SpeedySpec\Block\`. Composer names `speedyspec/block-domain`, `block-app`, `block-infra-memory`,
  `block-contract-tests`.
- Built and tested standalone with Pest, as the SEO packages were.
- Its own monorepo, with the directory layout of `SpeedySpec/speedyspec-api-seo` (`packages/<name>`, path
  repositories), at `SpeedySpec/speedyspec-api-block`, cloned locally to `packages/speedyspec-api-block/`.

## Approaches considered

1. **Chosen: a neutral data model with blocks and builders separated.** The project owner picked the neutral model,
   required that builders be separate from blocks, and placed a block's HTML output with the plugin that adds it.
2. **Mirror WordPress's model.** Rejected: WordPress's quirks would become everyone's contract.
3. **Typed component classes.** Rejected: a block could not be defined by data alone.

The research supports option 1: 98.6% of the 5,166 settings declared by the compared plugins are already plain
structured data. The remaining cost, translating WordPress blocks whose settings live in saved HTML, stays in the
WordPress adapter.

## The three roles

| Role | Carries | Runtime dependencies |
|---|---|---|
| **Block package** | Block types, their schemas, their behaviour, and their output. | `block-domain` only |
| **Builder** | The UI for arranging and configuring blocks and building block templates. | the builder contracts only |
| **Host** | Chooses block packages and a builder, stores documents, supplies data sources. | anything |

A block package never contains builder UI, and a builder never contains block-specific code. A block package has
one entry point, a provider class implementing a domain contract, through which it registers everything it offers.
That is how a host "chooses" a package and how a test loads one.

**A block's output ships with the block.** The plugin that adds a block also registers how it is drawn as HTML. That
output goes through a contract and is bound per block type, so a theme may replace the output of one block without
touching the block, but nobody has to supply output separately for a block to appear on a page.

**How portability is enforced.** Composer cannot forbid a dependency, so the shared contract suite a block package
runs does it: it reads the package's `composer.json` and fails if its runtime `require` lists anything besides PHP
and `speedyspec/block-domain`; it checks that every block type has catalogue metadata, at least one example, and
front-end HTML output; it resolves and draws every example in both modes with no builder present; and it checks
that no private setting or admin context value from an example appears in front-end output.

## The model

### Block type

- **Name:** `vendor/name`, lower-case.
- **Version:** an integer, starting at 1, raised when the meaning of stored settings changes.
- **Settings schema:** a JSON Schema object, including defaults. A setting may be marked private: it is never part
  of anything a visitor receives, but it is available in admin mode so an editor can change it.
- **View schema (optional):** a JSON Schema object for the view this type produces. When declared, views are
  checked against it and builders may rely on it to draw their own live previews.
- **Children policy:** none, any, or a list of allowed block type names.
- **Parent policy:** any, or a list of block type names this type may be placed directly inside. (11 of 14 plugins
  restrict child blocks to a parent this way.)
- **Provides:** names (namespaced, `vendor/key`) mapped to keys of this type's view. Descendants receive those values.
- **Uses:** names this type wants from its ancestors.
- **Assets:** names of scripts and styles the block needs, as opaque identifiers the host resolves.
- **Examples:** at least one set of example settings, used by the contract suites and by builders for previews.
- **Catalogue metadata:** title, description, category, keywords, and an icon as an opaque identifier. This is what
  a builder shows when listing blocks.

Schemas may carry annotations a builder uses to draw controls: `title` and `description` per setting, plus `x-control`
(a hint naming the kind of control) and `x-group` (which panel the setting belongs to). A builder that does not
recognise a hint falls back to a control chosen from the setting's type.

Creatable from a plain array, so a JSON file can define one.

### Block

- **Id:** assigned by the system and unique within its document. Saving and loading never change ids. They change
  only through an explicit duplicate operation, which hosts call when copying or importing blocks. (9 of 14 plugins
  invent an instance id; none regenerates it on copy, and only 3 cope with the duplicates when rendering.)
- **Type name and type version.** A document may hold a block whose type is not registered.
- **Settings:** stored as given; defaults apply when resolving.
- **Extensions:** settings added by decorators, stored per namespace, apart from the block's own settings.
- **Bindings:** a setting path mapped to a data source name, arguments, and an optional fallback. A path addresses a
  top-level or nested setting, and may name a decorator namespace to address an extension setting.
- **Children:** an ordered list.

Blocks and trees are immutable; a change returns a new tree. A tree can find a block by id and walk its blocks.

### Document

A tree with an id, a kind (`page`, `post`, `template`, `part`, or a host-defined kind), and a format version. A
document repository loads a document by id from storage the host trusts.

### Decorator

A named group of settings, with its own schema, attached to all block types or to a named list of them. Its values
live in the block's extensions under the decorator's namespace. A decorator may also declare a view schema for the
values it contributes to rendering. This replaces patching block registration, which is how the compared plugins add
conditions, animation and custom CSS to other blocks.

### Data source

A named supplier of values. It declares a schema for its arguments and a schema for the value it returns. Given a
binding and the context it returns a value or reports none. Bound values are data, never markup.

A bound value is checked against the schema of the setting it targets. The first valid candidate wins, in this order:
the data source's value, the binding's fallback, the stored setting, the schema default. A rejected candidate records
an issue but does not hide the block, so a visitor's out-of-range value cannot make a form field vanish.

A private setting cannot be bound. A binding that targets one is ignored and recorded as an issue, so visitor
input can never reach something like a notification recipient.

### Context

Values passed down the tree under namespaced keys, plus a typed current-item entry (item, index, collection id) for
loops, so no block reads global state. A block's provided values are taken from its view.

### Mode

Every resolve and render call names a mode.

- **`front`:** what a visitor gets. Private settings and admin context are never included in a view or a client
  export. A resolved block's settings are server-side and complete in both modes (forms read a notification
  recipient from them at submission time); only the view and the client export are visitor-safe.
- **`admin`:** what an editor gets. The host decides who may ask for it; the block system does not check
  permissions.

In admin mode:

- A block's view may carry an **admin context**: extra values for the person editing, such as the list of options
  a setting can take, a placeholder for empty content, or a note about where a value comes from. The block's
  behaviour supplies it.
- The system adds, for every block, what a builder needs without block-specific code: the block's issues, which
  settings are bound and to what, and whether a gate step would hide the block on the front end.
- A block that a gate step would hide is still resolved and drawn, marked as hidden, so it can be edited.
- Private settings are included.
- All of this is carried on the resolved block itself, not on its view, so a block that stopped before producing
  a view (invalid settings, a failed behaviour, an unknown type) still has its admin context.

### Behaviour and view

A block type may have a behaviour: logic that turns the block (its id, children, extensions and bindings), its
resolved settings, context and mode into a **view**, a plain structure of values with no markup. A type with no behaviour has a default view: its resolved settings minus
private ones. A query block's behaviour runs the query; a form block's behaviour knows its fields. The view is
what the block's own output draws, and what a builder may use to draw a live preview.

A view may ask for its children to be produced once per item, giving the current-item context for each pass. By
default children are produced once. This exists in phase 1 so that loops, which arrive in phase 4, do not change
the contracts.

### Output and attributes

An output renderer turns one block's view, plus the already-produced output of its children, into output for one
target and one mode. Phase 1 has one target, `html`. It never sees stored settings.

Each block package registers the front-end output for each of its block types, and may register a separate admin
output that shows the extra options and context. When a block type has no admin output, its front-end output is
used in the admin. Each registration is one binding per target, mode and block type, so a host, theme or builder
can replace a single block's output in either mode.

Decorators affect output through a structured **attribute bag** (classes, data attributes, inline style
declarations, an element id), never by editing markup. The package that adds a decorator may register a decorator
output step that reads the decorator's values and adds to the bag. The block's output renderer receives the bag and
applies it to its root element.

Escaping is a domain contract: one escaper per output target, with named contexts (text, attribute value, URL). The
default `html` escaper lives in `block-app`; every output renderer receives it. A block package therefore needs
nothing but the domain to escape correctly.

### Block template

A named, saved arrangement of blocks: a title, description, category and a tree fragment. A template repository
lists, loads, saves and removes templates. Inserting a template into a document duplicates its blocks, so they get
new ids.

### Serializer and client export

The serializer turns a document into a string and back. Phase 1 ships JSON. Loading never needs block types to be
registered, and a block of an unknown type round-trips unchanged, private settings included.

A separate client export produces the copy of a document that may leave the server, for a given mode. It needs the
block type registry. For `front` it removes every private setting, and for a block whose type is not registered, or whose stored type
version differs from the registered one, it removes all settings and bindings, since it cannot know which are safe. For `admin` it keeps everything.

## What a builder uses

A builder talks only to these contracts. All of them work on any block.

- **Catalogue:** list the available block types with their catalogue metadata, optionally narrowed to those that
  may be placed at a given spot in a document.
- **Placement:** ask whether a block type, or an existing block, may go at a given position. The answer combines the
  target's children policy and the block's parent policy.
- **Editing operations:** insert, move, remove, duplicate, change settings, change a decorator's settings, set or
  clear a binding, and insert a block template. Each takes a document and returns a new document or a refusal with
  reasons. An operation that would break a placement rule or produce settings that fail the schema is refused, so a
  document saved through these operations is always valid.
- **Preview:** for a block in a document, get its view and its output for a target in admin mode, with its admin
  context and issues; or in front mode, to see what a visitor will see.
- **Templates:** list, load, save and remove block templates.
- **Documents:** load and save through the document repository; the client export in admin mode gives a builder the
  full document.

These are PHP contracts with no transport. Exposing them over HTTP for a JavaScript builder is a later phase.

## Resolving and rendering

Resolving produces, for every block, its final settings, its view and its context. Rendering resolves and then
draws. Both walk the tree depth first through the same stages. Each stage has a fixed core step that cannot be
removed, and a typed extension point that packages may add steps to. A step sees only what its stage allows.

| # | Stage | Fixed core step | An added step may |
|---|---|---|---|
| 1 | **Prepare** | Look up the type; run registered migrations from the block's stored type version to the current one; overlay defaults. | Change the stored settings and extensions it was given. |
| 2 | **Gate** | None. | Decide the block is not shown. It sees prepared settings and context; it changes nothing. In admin mode the decision is recorded and the block continues. |
| 3 | **Resolve** | Apply bindings as described above. | Nothing; data sources are the extension point. |
| 4 | **Validate** | Check settings against the schema and children against the children policy. | Add issues. |
| 5 | **View** | Run the behaviour or build the default view, with admin context in admin mode; check it against the view schema when one is declared; compute provided context. | Add decorator view values under the decorator's namespace. |
| 6 | **Render** | Build the attribute bag; produce children; call the block type's output renderer for the target and mode, falling back from admin to front. | Add to the attribute bag. |
| 7 | **Collect** | Record the block's declared assets. | Add asset names. Output is not changeable here. |

Gate runs before Resolve so that a hidden block's data sources are never called. Consequently a decorator's gating
settings are read as stored and cannot be bound.

Resolving runs stages 1 to 5; rendering adds 6 and 7. Neither throws for bad content: a migration, step, behaviour
or data source that throws is caught and recorded as an issue. One resolve call handles at most a set budget of
blocks (50,000 by default), so nested repeating blocks cannot multiply without bound. An unknown type, unknown data
source, invalid settings, disallowed child, invalid view or missing output renderer records an issue (block id,
kind, message). The block then yields nothing, with two exceptions: a rejected bound value falls back as described,
and a block with no output renderer still passes its children's output through. The result of rendering is the output, the issues, and
the assets; the result of resolving is the resolved tree and the issues.

## Package layout

```
packages/speedyspec-api-block/packages/
├── block-domain/          Contracts, value objects, entities, exceptions, service container. Zero dependencies.
├── block-app/             Framework-free defaults: registries, schema validator, binding resolution, the stage
│                          pipeline, document resolver and renderer, editing operations, catalogue, placement,
│                          preview, JSON serializer, client export, html escaper.
├── block-infra-memory/    In-memory document and template repositories and data source, for tests and
│                          storage-less hosts.
└── block-contract-tests/  Shared Pest suites: one for block packages, one per replaceable contract. In part 1a it
                           requires only `block-domain` and Pest; the block-package suite in part 1b adds
                           `block-app` and `block-infra-memory`.
```

Dependency rules: the domain requires only PHP; `block-app` and `block-infra-memory` require only the domain;
services are resolved only through the container; `block-app` provides the virtual package
`speedyspec/block-app-implementation`. The contract-test package is a development tool and is the one package
allowed to require implementations, because running a behaviour needs the pipeline.

`block-domain` has its own `BlockServiceContainer`, behaving like `SeoServiceContainer`. This duplicates a class
across two monorepos; extracting a shared container package is the right fix but touches SEO, hooks and options, so
it is left out and noted here.

## Contracts (phase 1)

Names are indicative; the builder may adjust signatures while keeping one responsibility per contract.

| Contract | Responsibility | Default |
|---|---|---|
| `BlockPackageInterface` | A block package's entry point: registers its types, behaviours, output renderers, decorators, migrations. | supplied by block packages |
| `BlockTypeRegistryInterface` | Register, remove, look up block types. | app |
| `BlockBehaviourInterface` | Turn a resolved block and context into a view. Optional per type. | supplied by block packages |
| `BlockBehaviourRegistryInterface` | Map block type names to behaviours. | app |
| `SettingsMigrationInterface` | Move one type's stored settings from one version to the next. | supplied by block packages |
| `DecoratorRegistryInterface` | Register decorators; list those that apply to a type. | app |
| `SchemaValidatorInterface` | Validate a value against a schema; apply defaults. | app (subset validator) |
| `DataSourceInterface` | Supply a value for a binding in a context. | infra-memory ships an array-backed one |
| `DataSourceRegistryInterface` | Register and look up data sources. | app |
| `PrepareStepInterface`, `GateStepInterface`, `ValidateStepInterface`, `ViewStepInterface`, `RenderStepInterface`, `CollectStepInterface` | One added step in the named stage, with that stage's typed input and output. | none |
| `StepRegistryInterface` | Hold the ordered added steps per stage. | app |
| `OutputEscaperInterface` | Escape a value for one target and context. | app (`html`) |
| `BlockOutputInterface` | Draw one view for one target and mode. | supplied by block packages |
| `BlockOutputRegistryInterface` | Bind an output renderer per target, mode and block type; fall back from admin to front. | app |
| `DocumentResolverInterface` | Resolve a document without rendering; return the resolved tree and issues. | app |
| `DocumentRendererInterface` | Render a document; return output, issues and assets. | app |
| `DocumentSerializerInterface` | Turn a document into a string and back. | app (JSON) |
| `ClientExportInterface` | Produce the copy of a document that may leave the server. | app |
| `DocumentRepositoryInterface` | Load and save documents by id. | infra-memory |
| `BlockIdGeneratorInterface` | Produce block ids. | app |
| `BlockCatalogueInterface` | List block types for a builder, optionally by position. | app |
| `PlacementRulesInterface` | Say whether a block may go at a position. | app |
| `DocumentEditorInterface` | The editing operations; each returns a new document or a refusal. | app |
| `BlockPreviewInterface` | A block's view and output while editing. | app |
| `BlockTemplateRepositoryInterface` | List, load, save and remove block templates. | infra-memory |

Domain value objects and entities: `BlockName`, `BlockType`, `ChildrenPolicy`, `Block`, `BlockTree`, `Document`,
`SettingPath`, `Binding`, `Decorator`, `Context`, `CurrentItem`, `ResolvedBlock`, `ResolvedDocument`, `BlockView`,
`AttributeBag`, `Mode`, `AdminContext`, `RenderRequest` (target, mode), `RenderResult`, `Issue`, `ValidationResult`, `ParentPolicy`,
`CatalogueEntry`, `Position`, `EditResult`, `BlockTemplate`, `BlockPreview`.

### Schema support

The default validator supports: `type` (string, integer, number, boolean, array, object, null, and lists of these),
`enum`, `const`, `default`, `required`, `properties`, `additionalProperties`, `items`, `minLength`, `maxLength`,
`pattern`, `minimum`, `maximum`, `minItems`, `maxItems`, and the annotations `title`, `description`,
`x-private`, `x-control` and `x-group`. A schema using any
other keyword is rejected when it is registered, so a rule is never silently ignored. A library-backed full JSON
Schema validator can replace it later as a separate infrastructure package.

## Phasing

Each phase gets its own plan; phases 2 to 7 each get a short design addendum first.

1. **Core (this design):** the four packages above, planned in three parts that each end in working, tested
   software.
   - **1a, model and resolving:** types, documents, ids, decorators, migrations, data sources and bindings,
     behaviours and views, stages 1 to 5, serialization, client export, the in-memory repositories.
   - **1b, output and the block package contract:** stages 6 and 7, escaping, the attribute bag, output renderers,
     the block package entry point, and the shared suite a block package runs.
   - **1c, builder contracts:** catalogue, placement, editing operations, preview, block templates.
2. **Builder API over HTTP:** a presentation package exposing the builder contracts to a JavaScript builder.
3. **Presentation vocabulary:** one responsive value shape and a breakpoint registry (all 14 plugins hard-wire
   different breakpoints), design tokens, a style-rule collector for block output, asset resolution across nested
   documents.
4. **Conditions and collections:** a condition registry (groups of conditions, no expressions) joining the Gate
   stage; a collection data source and a looping block using the current-item context.
5. **Structural blocks:** section, columns, column, grid and element as a block package sharing one settings
   vocabulary.
6. **Document references:** a block that includes another document (headers, reusable sections) with a guard
   against loops; a library provider contract for template collections.
7. **WordPress adapter:** `block-infra-wp` with a serializer for WordPress's markup, a bridge from its block type
   registry, and its bindings exposed as data sources.

**Forms** is a separate project that can start once parts 1a and 1b are done: form blocks need resolving at
submission time and their own output on the page. A builder UI is also a separate project and needs 1c and phase 2.

## What forms will need, and where phase 1 provides it

| Need | Provided by |
|---|---|
| Stable field identity independent of label and order | Block ids that only an explicit duplicate changes, plus a key setting on each field |
| The form definition taken from storage, never from the request | `DocumentRepositoryInterface` |
| Each field's final rules at submission time, without rendering | `DocumentResolverInterface` and finding blocks by id |
| A field learning which form it belongs to | Provides and uses, computed in the View stage, so available when resolving |
| A prefilled or resubmitted value that cannot break the field | Bindings with validated fallback |
| A notification recipient that a visitor never receives and cannot influence, but an editor can change | Private settings: absent in front mode, present in admin mode, not bindable |
| Field rules | Ordinary settings in the field block's schema |

## Left to the builder

- Exact method signatures and value object internals.
- The id format, provided ids are unique within a document.
- The JSON shape of a serialized document, provided it round-trips every field, carries a format version, and is
  documented in the serializer's package.
- The syntax of a setting path.
- How issues are categorised beyond the kinds named above.
- Internal structure of each package.

## Deferred from part 1a

Minor findings from the part 1a reviews that were left unfixed. The plans for parts 1b and 1c, and the forms design,
should check this list.

- **Before part 1b (output):** a decorator's view schema is checked at registration but not enforced when resolving;
  a gate step returns only yes or no, so admin mode cannot say which step hid a block; misplaced blocks are dropped
  from the resolved tree while unknown-type blocks stay in it with no view.
- **Before part 1c (builder contracts):** the resolver's own placement check duplicates what the placement contract
  will own, and is recomputed on every pass; the five stages are bound under their class names, so one stage can only
  be replaced by a subclass; an issue carries no pass or item identity.
- **Before forms:** `provides` keys are not required to be namespaced, and number-like keys are handled
  inconsistently; a view schema that requires a private key fails in front mode only; finding a block in a
  `BlockTree` (not the resolved document) is linear per lookup.
- **Before phase 4 (loops):** a negative resolve budget is not rejected; when the budget runs out, that block's admin
  context has no binding information.
- **Serialization:** a nested object whose keys are exactly 0..n-1 reloads as a list; integers beyond 64 bits become
  floats; unknown keys are dropped on save; an empty binding source is accepted. A front export keeps bindings
  (source names, arguments, fallbacks) and all decorator extensions, since only private settings must be removed;
  a decorator that later stores something sensitive needs its own rule.
- **Schemas:** dotted error paths are ambiguous when a property name contains a dot; a schema's `default` is not
  checked against the schema itself; a very deep schema uses a lot of memory during registration.
- **Contract suites:** the data-source suite does not compare front against admin results or check the unknown
  arguments against the source's schema.
- **Tests and style:** some global test helpers are unguarded or unused; three tests use a placeholder assertion for
  "does not throw"; about 80 docblock `@param` tags restate a type the signature already gives; eleven private
  helpers lack a summary line.
- **Across monorepos:** the service container is duplicated from the SEO repository; the SEO packages' brace style
  does not match the rules file.

## Deferred from part 1b

Minor findings from the part 1b review that were left unfixed, and the pressure points it named for part 1c.

- **Styles in the attribute bag:** legacy script vectors (backslash escapes, comments, `-moz-binding`, `behavior`) and
  `url(...)` to any host are accepted. Not exploitable in current browsers; decide the rule with the presentation
  vocabulary (phase 3).
- **Attribute bag hardening:** the class can be subclassed to bypass its checks, and the escaper prints attribute
  names from a bag as given; names are case-sensitive, so `data-x` and `DATA-X` can both be emitted.
- **Steps see everything:** render and collect steps receive the whole resolved block, private settings included, in
  front mode. They are trusted code, like outputs, but outputs are confined to the view and steps are not.
- **`RenderResult` is server-side:** it carries the resolved document with complete settings; say so in its docblock.
- **Issue messages:** an `InvalidAttribute` issue echoes the offending class, name or id.
- **Renderer edge cases:** an empty asset name is accepted; children's assets are kept when the parent's output
  throws; render steps run for a block with no output; an output that echoes bypasses the result; issue order mixes
  pre-order and post-order.
- **What the vendor rule covers:** only block type definitions added through the loader. Any package may replace any
  type's behaviour, output or migrations and add any decorator, data source or step; a type added straight to the
  registry is outside the rule.
- **The block package suite:** it is tied to the default application and in-memory layers, reads only `require`, and
  exercises each type's first example without bindings, context values or children.
- **Tests and docs:** one renderer test's title promises more than it checks; the new bindings are missing from the
  provider's "every contract resolves" test; some docblock tags restate types; the example package's admin output
  does not show the use of the admin context.

### For part 1c

- **Previewing one block or a fragment** has no explicit method. It works today by resolving the document and drawing
  a resolved document whose only root is that block; make it explicit or document it.
- **One output string per render.** A builder that swaps one block after an edit must draw the document again.
- **Unresolved blocks draw nothing in admin mode,** so a canvas has nothing to select for a broken block. A
  placeholder output is a new seam.
- **Package ownership:** the loader lists a package's types but has no lookup from a type to its package, and records
  nothing for decorators, data sources, outputs or steps.
- **Block templates from packages:** the registrar has no way to register one.

## Deferred from part 1c

Minor findings from the part 1c review that were left unfixed.

- **Id generator that keeps colliding:** `insert` and `duplicate` throw `RuntimeException`. It is a broken
  collaborator, not a bad request; say so in `DocumentEditorInterface`.
- **`move` indexes:** the position is read with the block already taken out, so moving to the end of the same parent
  uses the count minus one. Documented and pinned; add a worked example to the docblock.
- **Template names across packages:** two packages of one vendor can register the same template name and the second
  replaces the first. Templates are also written outside the rolled-back part of a package load.
- **Unregistered parent:** the catalogue offers nothing inside a block of an unregistered type, while the editor
  accepts an insert there. The catalogue's `placeableAt` also ignores whether the index exists.
- **Empty paste:** `insertBlocks` with no blocks is accepted and changes nothing.
- **`updateSettings` stamps the current type version** without migrating bindings or decorator settings stored under
  the old one.
- **`BlockPreview` is server-side:** its resolved block carries complete settings, private ones included, in front
  mode too; say so in its docblock before it is sent anywhere.
- **Misplaced blocks in admin mode:** they are left out of the resolved tree, so they get no placeholder; a preview
  reports the reason but draws nothing.
- **Docblocks:** implementations of the new contracts have no `{@inheritDoc}`; three private helpers in the editor
  have no summary; `DocumentResolver`'s constructor docblock does not mention the placement contract; a blank line
  is missing before `BlockTree::insert()`'s docblock.

## Addendum: storage

Added after phase 1, at the user's request: "build a contract that allows a Repository to store the documents and
saved templates."

- **One storage contract, `RecordStoreInterface`,** in `block-domain`: named text records in collections, with
  `get`, `put`, `delete` and `keys` (first-put order). It is the only thing a host implements to use its own storage.
- **Repositories are written once,** in `block-app`: `StoredDocumentRepository` and `StoredBlockTemplateRepository`
  serialize through `DocumentSerializerInterface` and the new `BlockTemplateSerializerInterface` and keep the text
  in a record store. `StorageServiceProvider` binds both. The repository contracts are unchanged, so a host can
  still replace a repository outright.
- **Record stores:** `block-infra-memory` (reference), `block-infra-filesystem` (one file per record, named by the
  hash of the key, atomic writes under a per-collection lock) and `block-infra-pdo` (one table, portable SQL).
  All run the shared `recordStoreContract` suite.
- **A stored record that cannot be read is an error,** never "not found", so damaged content is not silently
  overwritten by a new empty document.
- **Not included:** listing or searching documents, revisions, optimistic locking against concurrent edits, and
  migrations between record stores. Listing documents is decided with the HTTP phase, which is its first user.

## Addendum: phase 4, conditions and collections

- **Conditions:** `ConditionInterface` (a name, an arguments schema, `holds()`) and `ConditionRegistryInterface` in
  `block-domain`. A condition is the only way a rule reaches outside the block system; there is no expression
  language.
- **Visibility:** a decorator, `core/visibility`, that applies to every block type. Its settings are `groups`: a
  list of groups, each a list of rules `{condition, arguments, negate}`. A block is shown when there are no groups,
  or every rule of at least one group holds. It joins the Gate stage, so a hidden block's data sources are never
  called, and the Validate stage, where an unknown condition or refused arguments become issues an editor sees.
- **Fail closed:** a rule that cannot be judged (unknown condition, refused arguments, a condition that throws)
  does not hold, even when negated. A mistake hides content rather than exposing it.
- **Collections:** `CollectionSourceInterface` (a name, an arguments schema, `items()`) and its registry. A
  collection source supplies the items a block repeats over; a data source still supplies single values.
- **The repeating block, `core/repeat`:** settings `source`, `arguments`, `limit` (1 to 500, default 50). Its
  behaviour asks the collection for items and returns one child pass per item, using the contract phase 1 reserved
  for this. It adds no markup. In admin mode a repeat with no items produces its children once, so they can still
  be edited. A collection is never called with arguments its schema refuses.
- **Reading the item:** the `core/current-item` data source (argument `field`, a dotted path) and the
  `core/current-item-is` condition (`field`, `value`, compared strictly).
- **Packaging:** all of it is `FlowPackage`, an ordinary block package in `block-app`, loaded by the host. The two
  registries are bound by the application provider.
- **Not included:** paging a collection, conditions on dates or users (they need the host), and a condition that
  throws being reported as an issue (it only hides the block).

## Addendum: phase 3, presentation vocabulary

- **Breakpoints:** `Breakpoint` (name, minimum width) and `BreakpointRegistryInterface`. Mobile-first: a value's
  base applies everywhere, each override from its breakpoint upward. Defaults `tablet` 768 and `desktop` 1024.
- **Responsive value:** one stored shape, `ResponsiveValue`: a single value, or a map of `base` and breakpoint
  names. A key that is not a registered breakpoint is ignored when drawing.
- **Design tokens:** `DesignTokenRegistryInterface`, names like `color.primary`. A setting refers to a token as
  `token:color.primary`; it is written as `var(--color-primary)`, and the tokens as custom properties on `:root`.
- **Style rules:** `StyleRule` (declarations, optional breakpoint) checks itself: only a small character set and
  the functions `var`, `calc`, `min`, `max`, `clamp`, `rgb`, `rgba`, `hsl`, `hsla`. No URL, quote, escape, comment
  or anything that ends a declaration can be constructed.
- **Collector:** `StyleCollectorInterface`. `add()` returns a class derived from the rules, so blocks styled alike
  share one rule; `css()` writes tokens, base rules, then one media query per breakpoint narrowest first. The host
  reads it after drawing and calls `clear()`. Collection happens only for the `html` target.
- **Vocabulary:** the `core/style` decorator, applied to every block type by `PresentationPackage`, with thirteen
  settings, each responsive and each with a strict format enforced by the schema, so the editor refuses anything
  else. A stored value that does not fit is dropped when drawing.
- **Not included:** hover and other states, arbitrary CSS, fonts, an unknown-token check, and asset resolution
  across nested documents, which moves to phase 6 with document references.

## Addendum: phase 5, structural blocks

- **A block package, `block-structure`** (`core/structure`), depending only on `block-domain`: `core/section`,
  `core/columns`, `core/column`, `core/grid`, `core/element`.
- **Tags come from fixed lists,** enforced by the schema and checked again by the output, so no stored value can
  name an element such as `script`.
- **Layout is written as style rules** through `StyleCollectorInterface`: flex for columns (stacked below a named
  breakpoint, in a row from it; an empty name means always in a row), a 1 to 12 share per column, and a 1 to 12
  column count for the grid, both responsive. Every value written is a bounded whole number or a fixed word.
- **Appearance is not theirs:** spacing, colour and the rest come from `core/style`.
- **Not included:** a wrap or order setting for columns, grid row and column spans for children, and a heading or
  paragraph block with rich text; `core/element` holds plain text only.

## Deferred from the storage review

One critical and nine important findings were fixed (lost saves on a failed commit, writers racing, unreadable
storage read as empty, interrupted writes, provider order, unreadable documents, the MySQL character set). Left:

- **Key and stored name can differ** when a record is written behind the repository's back; the repository returns
  the inner id or name.
- **Filesystem:** exception messages contain server paths; directories are created `0775` and files take the
  umask; writes are not flushed to disk before the rename; each new key rewrites the whole index; record file names
  are the unsalted hash of the key.
- **Database:** no index on `(collection, position)`; a reserved word passes as a table name and fails in SQL;
  `install()` commits an open transaction on MySQL; an undecodable stored key lists as an empty key; only SQLite
  has been run, MySQL, MariaDB and PostgreSQL were judged by reading.
- **`save()` can throw `InvalidDocumentException`,** which `DocumentRepositoryInterface` does not declare.
- **A missing record store** is reported on first use of a repository, not when providers are registered.

## Addendum: phase 6, document references

- **`core/include`:** a block with one setting, `document`, the id of a stored document. Its output asks
  `DocumentIncluderInterface` to draw that document with the ordinary renderer, in the same mode and with the
  including block's context, so an included card inside a repeat sees the current item.
- **Guards:** a document already being drawn further up is not drawn again; inclusions nest at most 8 deep; one
  inclusion from a page leads to at most 500 others. A stopped inclusion draws nothing and records a
  `document-loop` issue.
- **What comes back:** output is only markup, so the includer keeps the issues and assets of what it drew. Assets
  are added to the page's result through a collect step; issues are read from the includer by the host, which then
  calls `clear()`. A document that is not stored is an `unknown-document` issue on the including block.
- **Admin mode:** the included output is wrapped in an element naming the document.
- **Template libraries:** `BlockTemplateLibraryInterface` (name, title, `all()`, `find()`) and its registry, with
  two libraries by default: templates defined in code and templates people saved. A remote collection is another
  implementation.
- **Not included:** editing an included document in place, per-inclusion overrides of its settings, and caching.

## Addendum: phase 2, builder API over HTTP

- **A presentation package, `block-presentation`,** depending only on `block-domain`. `BuilderApi::handle()` takes
  an `ApiRequest` (method, path, query, raw body) and returns an `ApiResponse` (status, body). No framework and no
  PSR dependency; a host adapts its own request in a few lines.
- **Routes** cover the catalogue, a vocabulary (breakpoints, tokens, conditions, collections, decorators),
  documents (list, create, read, delete), editing operations (one endpoint, the operation named in the body),
  preview, render, and templates (list by library, save a block as one, remove).
- **Permissions:** `BuilderAccessInterface::allows(BuilderAbility, ?documentId)` in `block-domain`, with abilities
  `view`, `edit` and `manage-templates`. Access is asked before a document is loaded, so a refusal does not reveal
  whether it exists. Reading a document for editing shows private settings, so it needs `edit`. With no rule
  bound, everything is refused.
- **Errors:** a refused edit is `422` with every refusal; a request the model refuses is `400` with the model's
  message; anything unexpected is `500` with no detail.
- **Added to existing contracts:** `DocumentRepositoryInterface::ids()` and `DecoratorRegistryInterface::all()`.
- **Not included:** authentication, request forgery protection, rate limiting, optimistic locking between two
  editors of one document, paging the document list, and a PSR-15 adapter.

## Addendum: phase 7, WordPress adapter

- **`block-infra-wp`,** depending only on `block-domain`. What it needs from WordPress comes through four ports
  (posts, block types, a renderer for dynamic blocks, bindings), each with a thin implementation that calls
  WordPress and can be replaced.
- **Markup:** `WpMarkup` parses and writes WordPress's block comments itself. Attributes become settings; the
  markup a block saved around its inner blocks is kept, in pieces, in a `wpContent` setting. Text outside any block
  becomes a `core/freeform` block; blank space between top-level blocks is dropped. Malformed markup is read as far
  as it makes sense.
- **Identity:** a block's id is `metadata.blockId` when present and not already used, otherwise its position
  (`wp-1-0`). Saving writes the id into every block's metadata, which changes a post's markup the first time it is
  saved through the adapter.
- **Bindings:** `metadata.bindings` is read into the block's bindings and written back.
- **Types and drawing:** `WpBlockTypeBridge` maps a WordPress type's attributes to a settings schema and its
  `parent`, `allowed_blocks` and context lists to policies. `WpBlockOutput` draws the saved markup with the
  children placed together at the first inner-block position, and passes dynamic types to WordPress.
- **Limits:** the WordPress-calling classes are not run against WordPress here; a binding changes a setting but
  not the saved markup of a static block, which is where WordPress shows it; a new document cannot be created by
  id, only opened by the id WordPress assigned; the saved markup is drawn unescaped, as WordPress does.

## Addendum: the reference builder

- **`block-builder`,** a JavaScript package with no dependencies and no build step, written only against the
  builder API. It is a reference: the proof that a builder needs nothing but the contracts.
- **Logic apart from interface:** an API client, document helpers, a form generator that turns a settings schema
  into fields, and a store that sends edits one at a time and shows the document or the refusals that come back.
  These are tested with Node's own test runner.
- **Interface:** a `<block-builder>` element with three panes: the document's structure (select, move up and down,
  indent, outdent, duplicate, remove, add at a place from what the server allows there), an inspector (a block's
  settings and each decorator's, from their schemas), and a live preview in a frame that can run no script.
- **Decided without the user:** the builder lives in the block repository rather than in the Astro admin app, uses
  no framework, and is a structure-and-inspector builder rather than one edited by clicking in the page. The
  interface was checked by hand in a browser against a local demo host.
- **Not included:** undo and redo, drag and drop, selecting in the preview, binding editing, a dedicated control
  for responsive values, saving templates, and embedding in the Astro admin.

## Deferred items closed after phase 7

- **Misplaced blocks in admin mode** are kept in the resolved tree, unresolved, so they are drawn as a placeholder
  and can be selected; a visitor's tree still leaves them out. (Parts 1a and 1c.)
- **A resolve budget below zero** is refused when the resolver is built. (Part 1a.)
- **Nothing can be placed inside a block whose type is not registered:** insert, paste and move refuse it as
  `unknown-type`, matching the catalogue. (Part 1c.)
- **Template names:** a package cannot register a template name that is already registered, and a failed package
  load removes the templates it added. (Part 1c.)
- **Docblocks:** `BlockPreview` says it is server-side; the editor contract explains broken collaborators, gives a
  worked example for `move`, and says what `updateSettings` does to the type version. (Part 1c.)

Everything else in the three deferred lists stays deferred.

## From the review of phases 4, 3 and 5

Fixed, each with a test that failed first:

- **Unbalanced brackets in a style value** are refused; one would have swallowed the rest of the stylesheet.
- **An empty visibility group** never holds, and the editor refuses one; it used to make a restricted block public.
- **A stored style value that does not fit** is dropped before the block is checked, by a prepare step, so the
  block keeps its place on the page. No issue is reported for the dropped value.
- **The repeat's check** ignores other block types that happen to have a `source` setting.
- **Columns** fall back to the narrowest registered breakpoint when theirs is not registered, and to a row when
  none is.
- **The editor runs the validate steps,** so a rule naming an unknown condition, or a repeat naming an unknown
  collection, is refused when edited instead of accepted and then hidden.

Decided, not fixed:

- **A client export is not gated.** It removes private settings only and carries hidden blocks as stored, because
  hiding needs the context a page is resolved in. This is now stated on `ClientExportInterface` and `Visibility`.
  A host that sends the front export of a document with visibility rules exposes what they hide.
- **Nested repeats can spend the resolve budget** and cut the page short. The block's description now says so;
  the resolver still stops the whole document rather than only the repeat.

Deferred minors: malformed rules reach data sources before validation hides the block; schema defaults in condition
and collection arguments are not applied; `color.primary` and `color-primary` name the same custom property; the
collector trusts a replaced token registry; `FlowPackage` registers its condition outside the registrar's commit, and
the registrar has no method for conditions or collections; an empty inner repeat in admin mode shows the outer item;
section and element schemas accept unknown settings; block children inside a `p` element; style and count values the
editor accepts but ignores; class sharing depends on declaration order; `ItemPath` reads arrays only; a collection is
not told the limit; no browser-level layout test.

## From the review of phases 6, 2 and 7

Fixed, each with a test that failed first:

- **Including a document was a way to read it.** The includer and the include block now ask the host's access rule
  (`view` on the front, `edit` in admin mode) before a document is included, and report a refused document exactly
  as a missing one.
- **`updateSettings` and `updateExtension` with no `settings`** are a bad request; they used to clear the block.
- **A template saved from a block** no longer carries the block's private settings.
- **A document that includes itself** is drawn once: the renderer names the page in the context and the includer
  treats it as already being drawn.
- **WordPress markup:** an attribute ending in a backslash survives a save; a block's saved markup is a private
  setting, so it can never be bound to a data source, and a binding on it in a post is ignored.
- **WordPress types:** a default that does not fit its attribute's type is dropped instead of invalidating every
  block of the type; a type name another package already registered is reported by `boot()`.
- **WordPress posts (by reading, not run):** an id must be a post number written the usual way, and only the
  listed post types outside the trash are documents.
- **The editor runs the validate steps for `setBinding`** as well.

Decided, not fixed:

- **Markup added through the API is drawn unescaped** unless the posts port cleans it on save. WordPress does; the
  port's contract now says an implementation must. The answer to an edit shows the markup as sent, which may differ
  from what WordPress stored after cleaning.
- **The WordPress adapter is labelled experimental.** Static blocks skip WordPress's render pipeline, dynamic blocks
  get no inner blocks or context, binding sources get no post context, and stored content that breaks a bridged
  schema or placement rule is not drawn. Each needs the ports to carry more and a live WordPress to check against.
- **Inclusion limits are per top-level inclusion,** not per page, and the includer's issues and assets grow until
  the host calls `clear()`. The API does; a host that draws documents itself must.
- **Empty objects nested in settings** become empty lists at the API and in WordPress markup, as the serializer
  already documents.

Deferred minors: invalid UTF-8 becomes `null` in a response; a model exception caused by stored data is answered as
400; array query values are a 500; creating a document on WordPress is a 500; document ids and pastes are
unbounded; issues, including setting paths, are sent to callers who may only view; included-document issues do not
say which document; a saved template can shadow a built-in name; very large or very deep WordPress markup; closers
that do not match are kept as text where WordPress closes the block; the attribute bag is ignored for WordPress
blocks; `ancestor` and `supports.inserter` are not bridged; every edit is a post revision; the adapter's ports are
passed by constructor rather than bound in the container; about fifty implementing methods have no docblock.

## Addendum: the builder's interface, redone

The first interface was rejected as hard to work with. It was redone after the editing model the compared plugins
share:

- **The page is the editor.** `block-presentation` adds a render step, `BuilderAnnotations`, that writes
  `data-block-id` on each block's element in admin mode only. The builder draws the page in a frame that allows
  same-origin access but no script, and listens for clicks from outside it.
- **A toolbar on the selected block** (select parent, move up, move down, duplicate, add, remove) replaces the row
  of buttons under the structure list.
- **A searchable add-block panel** by category, which states where the block will go and dims what the server does
  not allow there; an outline panel; a trail of ancestors.
- **Preview widths** taken from the breakpoint registry.
- **A tabbed inspector:** Block, Style (sections, a per-width switch, colour tokens, alignment buttons) and
  Visibility (a rule builder whose argument fields come from each condition's schema). A rule that is not finished
  stays on screen until the server accepts it.
- **Refusals beside the field** they are about.

Colour is used for one meaning only: teal marks what is selected or active.

## Addendum: drag and drop, undo and redo

- **Drag and drop** is the builder's alone: `src/drag.js` turns a drop into the position the existing `move` and
  `insert` operations take. The server decides, as always, whether the block may go there.
- **Undo and redo** needed one addition to the contracts: `DocumentEditorInterface::restore()`, exposed as the
  `restore` operation. It replaces every block of a document with the given ones, keeps their ids, and holds them
  to the same checks as a paste, since the state comes from outside and block types may have changed since it was
  taken. The builder keeps the blocks the server gave it before each edit (50 steps by default) and asks for one
  back; it stores no inverse of each operation.
- **What that means:** history lasts for the session and is not shared. Undoing restores the whole document, so
  it also discards anything another person changed in the meantime; there is still no locking between editors.
- **Fixed on the way:** pasted and restored blocks are now read from the request as objects, so a block's empty
  settings, extensions and bindings stay maps. Before, a paste of blocks that carried them was refused.
