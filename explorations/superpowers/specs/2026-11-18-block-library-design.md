# Block library and replaceable editor design

Date: 2026-11-18
Builds on: `2026-11-17-block-model-design.md` and its addenda (phases 1 to 7, the reference builder).
Repository: `SpeedySpec/speedyspec-api-block`.

## Intent

The block system has seven blocks. The 14 block plugins it was compared against ship 30 to 80 each, plus editing,
styling and data features the reference builder lacks. The user wants all of that available, but not baked in:

- **Blocks as shared definitions.** In the user's words, a plugin adding "its own take" on a block should "replace
  the output but keep the same settings, so pages still open". The definition of a block type (its name, settings
  and catalogue entry) is the contract; how it is drawn is a replaceable implementation.
- **Organised by area,** each area its own pair of packages, so a plugin can replace one area (say, the
  interactive blocks) without touching the others.
- **Grouped by plugin in the editor.** In the user's words, "each plugin should be grouped in a section so that
  they are organized and searchable".
- **A replaceable editor.** The editor is its own package; a site can turn it off, and another plugin can supply
  its own.

This follows the standing rule for the project: contracts in their own package, implementations separate and
replaceable in part or in whole, and nothing depending on WordPress.

## Goals

- Every block a page needs to be written, not only arranged: text, media, buttons and the common marketing,
  interactive and site blocks the plugins share.
- A page made with one plugin's version of a block opens, unchanged, with another's.
- A plugin can redraw one block, a whole area, or every block, by registering outputs. It never has to redefine a
  block type to do so.
- A site can run with no editor, the reference editor, or another plugin's editor, by changing one binding.
- The add-block panel shows a section per plugin, searchable across all of them.

## Non-goals

- **Forms.** Still its own project, built on this one. The block system already has what forms need (private
  settings, stored definitions); this design does not add form blocks.
- **A WordPress-compatible block library.** Block names and settings here are this system's own. The WordPress
  adapter keeps drawing WordPress's own blocks as it does now.
- **Pixel-matching any plugin.** The default outputs are plain, accessible markup with the shared style vocabulary.

## Anti-goals

- **A block whose settings depend on who draws it.** If two outputs need different settings, they are two block
  types.
- **Editor code inside a block package.** A block package carries a definition or an output, never editor code.
  Anything an editor needs to know about a block is declared in its definition, as data.
- **Arbitrary HTML or CSS from content.** Formatted text is a cleaned subset; styles stay within the vocabulary.

## Approaches considered

1. **Definitions and default outputs in two packages for the whole library.** Simple; a plugin replaces one block
   or all, but cannot leave out just one area's defaults.
2. **One package with both, plugins override outputs.** Fewest packages; the defaults always load, with their
   styles and asset names, even when every block is redrawn elsewhere.
3. **Split by area, each area a definitions package and a default outputs package.** Chosen by the user. A plugin
   can replace one block, one area or everything, and a site loads only the areas it uses. It costs more packages,
   and moving a block between areas later is a breaking change, so the area of each block is decided here.

For the editor: a host-side contract that says which editor opens a document was recommended over leaving each
host to load whichever script it likes, because only a contract gives one uniform way to turn the editor off or
swap it. The user did not object; it is treated as decided.

## Packages

All new packages require only `speedyspec/block-domain` at runtime, except where noted.

| Area | Definitions | Default outputs |
|---|---|---|
| Structure (exists) | `block-structure-domain` | `block-structure` |
| Text | `block-text-domain` | `block-text` |
| Media | `block-media-domain` | `block-media` |
| Actions | `block-actions-domain` | `block-actions` |
| Interactive | `block-interactive-domain` | `block-interactive` |
| Site | `block-site-domain` | `block-site` |

Naming follows the project's layer names: the definitions are the contract (`-domain`); the default outputs are one
implementation of it. *My call:* names; the user may rename.

- **A definitions package** is a block package that registers block types (names, settings schemas, catalogue
  metadata, examples, children and parent policies, provided and used context), the behaviours that compute a view
  from settings, and any migrations of its settings. It registers no outputs. Behaviours sit with the definition
  because a view's shape is part of what every output relies on.
- **A default outputs package** is a block package that registers one front output (and, where useful, one admin
  output) for each type of its area, and the assets those outputs need. It declares the definitions package as a
  dependency, which the block package suite allows for `-domain` packages.
- **A plugin's own take** is a block package that registers outputs for some or all types of an area. It depends on
  the definitions package, not on the default outputs package.
- **`block-structure` is split** into `block-structure-domain` and `block-structure`, to match. The block names and
  settings do not change, so stored pages are unaffected.

### Who may register an output

Today any package may register an output for any block type (recorded under "What the vendor rule covers" in the
part 1b deferrals). That stays, because it is exactly what a plugin's own take needs. Two rules are added so the
result is predictable:

- **Load order decides.** The last package to register an output for a type, mode and target wins. A host loads
  default outputs first and replacements after.
- **The loader records who drew what.** `BlockPackageLoaderInterface` gains `outputOwner(BlockName, Mode, target):
  ?string`, so a builder or a site health screen can show which package draws a block.

### Who owns assets

Assets belong to whoever draws, not to the definition, so a plugin's own take on tabs brings its own script and the
default one stops loading. *My call, from the builder check.*

- **Library definitions declare no assets.** `BlockType` keeps its `assets` field for packages that both define and
  draw, such as the example package.
- **Outputs declare theirs.** A new optional interface, `DeclaresAssetsInterface` (`assets(): list<string>`), is
  implemented by an output that needs assets. The Collect stage adds them for blocks drawn by that output, after the
  type's own.
- **Replacing an output replaces its assets.** Since the replaced output is no longer asked, its assets are not
  collected.
- **Asset names are namespaced** (`vendor/name`) and owned by the package that registers them.

### Required outputs

A definitions package with no outputs fails today's block package contract suite ("gives every block type…
front-end output"). The suite gains a mode for definitions packages that checks shape, examples and schema but not
output, and a mode for outputs packages that loads the definitions package first and checks every type of it has
front output, escapes text and keeps private settings out. *My call.*

## Grouped by plugin

- **A section per package.** The add-block panel groups block types by the package that **defined** them, not by
  category. A plugin that only redraws blocks adds no section.
- **Package details.** `BlockPackageInterface` gains `title(): string` and `description(): string`, so a section
  has a name a person understands. This is a breaking change to the interface, acceptable at 0.1.0. *My call:* a
  required method rather than an optional metadata object, because every section needs a title.
- **The catalogue carries it.** `CatalogueEntry` already has `package`; it gains `packageTitle`. The builder API's
  catalogue lists packages once with their title and description, and each entry names its package.
- **Search runs across every section** and matches a block's title, keywords, description, and its package's
  title. A section with no match is hidden. Within a section, blocks keep their category as a subheading when a
  package uses more than one.
- *Interpretation, mine:* "each plugin should be grouped in a section" is read as the add-block panel. The outline
  and inspector are not grouped by plugin.

## Formatted text

Writing a page needs formatted text that every editor writes and every output prints.

- **The stored form is a small HTML subset,** held as a string in an ordinary setting.
- **Allowed elements:** `strong`, `em`, `s`, `code`, `sub`, `sup`, `mark`, `br`, and `a` with `href` only.
- **Allowed links:** `http`, `https`, `mailto`, `tel`, and paths on the same site.
- **Everything else is removed:** any other element (its text is kept), any other attribute, and comments.
- **Declared in the schema:** a setting is formatted text when its schema says `"type": "string"` and
  `"x-control": "rich-text"`.
- **Cleaned on every way in.** A new contract, `RichTextSanitizerInterface` in `block-domain`, with an
  implementation in `block-app`, cleans the value:
  - **when edited:** the document editor cleans formatted-text settings before checking and storing them, so what
    is stored is always clean;
  - **when drawn:** outputs call it again, through a helper on `RenderInput`, because stored content may predate the
    rule or arrive by another route.
- **Bound values are plain text.** A binding into a formatted-text setting is escaped as plain text, never treated as
  markup, keeping "bound values are data, never markup".
- *My call:* an HTML subset rather than a structured format (such as a tree of text runs and marks), because every
  editor can produce it from in-page editing and every output can print it, at the cost of needing the sanitizer on
  both edges. The user saw this call and did not object.

## Editing on the page

For text to be typed where it appears, an editor must know which element holds which setting.

- **Outputs mark editable settings while editing.** `RenderInput` gains `editable(string $setting): AttributeBag`
  that adds `data-block-setting="text"` to an element when the request's `editing` flag is set (see the editor
  seam). A visitor's page never carries it.
- **Only formatted-text and plain-text settings are marked.** Other settings stay in the inspector.
- **The editor does the rest.** It makes a marked element editable and sends the result as an ordinary
  `updateSettings`. The server cleans it as above.
- A block whose output does not mark its settings is still edited in the inspector, as now.

## Media

Image, gallery, video and audio blocks need files, and files belong to the host.

- **A new contract, `MediaLibraryInterface`, in `block-domain`:** find an item by id, list or search items, and
  accept an upload.
- **An item** has an id, URL, kind (image, video, audio, file), MIME type, width and height where known, alt text,
  and a list of sizes (name, URL, width, height).
- **Blocks store the media id**, not the URL, so moving files does not break pages. The URL is looked up when the
  block is resolved, by the block's behaviour, through the library.
- **The builder API** gains `GET /media`, `GET /media/{id}` and `POST /media`, behind a new
  `BuilderAbility::UploadMedia` for uploads.
- **Uploads,** *my call:*
  - the file travels as the raw request body, with its name in the query and its type in `Content-Type`, so
    `ApiRequest` gains headers;
  - accepted kinds are JPEG, PNG, GIF, WebP, AVIF, MP4, WebM, MP3, Ogg, WAV and PDF, checked by content, not by
    name; SVG, HTML and anything else is refused;
  - the size limit is the host's, 20 MB by default.
- **Implementations:**
  - `block-infra-memory` gains one for tests;
  - `block-infra-filesystem` gains one that stores uploads in a directory;
  - `block-infra-wp` gains one over WordPress attachments.
- **A media id that no longer exists** draws nothing for a visitor and shows a placeholder in admin mode, with an
  issue.

## Front-end assets

Interactive blocks (tabs, accordions, sliders, countdowns) need scripts and styles on the visitor's page. Block
types already declare asset names; nothing maps a name to a file.

- **A new contract, `AssetRegistryInterface`, in `block-domain`:** register an asset by name with a kind (script or
  style), its source, and the names it depends on.
- **A source is inline content, a full URL, or a file inside a package** (package name and a path relative to it). A
  package only knows its files' paths, not where the host serves them.
- **The host turns package files into URLs** through a second contract, `AssetLocatorInterface`
  (`url(string $package, string $path): string`). *My call, from the builder check.* The demo host serves the
  files of the packages it loads; a host with a build step can map them to its own copies. The reference editor's
  own scripts are package files resolved the same way.
- **Resolution:** `resolve(list<string>)` gives them back in dependency order, each once.
- **Who registers what:** an outputs package registers the assets its outputs name. A plugin's own take registers
  its own.
- **Who places them:** the host places resolved assets in the page. The renderer's result already lists asset
  names.
- **No JavaScript is needed for content to be readable.** Interactive blocks draw complete, usable markup (an
  accordion is a set of `details` elements; tabs show every panel in order); scripts only enhance it.

## The editor seam

- **A new contract, `EditorInterface`, in `block-domain`:**
  - `name(): string`
  - `title(): string`
  - `open(EditorRequest): EditorPage`
- **`EditorRequest` carries** the document id, the URL the builder API is mounted at, and a set of host values,
  such as a request-forgery token.
- **`EditorPage` carries** the asset names to load, resolved through the asset registry and locator like any other
  assets, and the markup to place. The host puts them in its admin screen.
- **Binding decides:**
  - The host binds zero or one `EditorInterface`.
  - With none, the host shows no editor; documents can still be drawn and changed through the API.
  - A plugin that brings its own editor binds its own.
  - *My call:* one editor per site, not one per document kind.
- **`block-builder` becomes a mixed package:**
  - its JavaScript, as now;
  - a PHP part with a `ReferenceEditor` implementing the contract and a service provider that binds it;
  - its `composer.json` requires only `speedyspec/block-domain`.
- **Editing marks follow the editor.** `RenderRequest` gains an `editing` flag. The builder API sets it on admin
  renders and previews only when an `EditorInterface` is bound. Both kinds of mark depend on it:
  - `BuilderAnnotations` (block ids) adds `data-block-id` only when the flag is set;
  - `RenderInput::editable()` returns an empty bag unless the flag is set.
  A site with no editor still gets admin renders and previews, without marks. *My call, from the builder check.*
- **What an editor must handle** is defined by the builder API and the block definitions as data, never by block
  package code. An editor that meets an `x-control` it does not know falls back to a plain field.

## The blocks

The area of each block is fixed here. Names are `core/…` like the existing system blocks.

- **Text:** `core/heading` (level 1 to 6, formatted text), `core/paragraph` (formatted text), `core/list` and
  `core/list-item` (ordered or not, nested), `core/quote` (text, citation), `core/code` (plain text, language
  label), `core/table` (header row, rows of formatted-text cells, caption), `core/separator`, `core/spacer`
  (height, responsive).
- **Media:**
  - `core/image` (media id, alt text override, caption, link, size);
  - `core/gallery` (media ids, columns responsive, caption);
  - `core/video` and `core/audio` (media id, controls, autoplay muted, loop, poster for video);
  - `core/embed` (URL from an allow-list of providers, drawn as a link plus an `iframe` with `sandbox`);
  - `core/icon` (name from an icon set the outputs package declares, label);
  - `core/icon-list` and `core/icon-list-item`;
  - `core/map` (address or coordinates, drawn as a link plus an embed through `core/embed`'s allow-list).
- **Actions:** `core/button` and `core/buttons` (text, link, style variant from a fixed list, size),
  `core/call-to-action` (heading, text, buttons as children), `core/testimonial` (quote, name, role, media id),
  `core/pricing-table` and `core/pricing-plan` (name, price, period, features, button, highlighted),
  `core/rating` (value, out of), `core/social-links` and `core/social-link` (service from a fixed list, URL).
- **Interactive:** `core/accordion` and `core/accordion-item` (title, children, open by default),
  `core/tabs` and `core/tab` (label, children), `core/slider` and `core/slide` (children, autoplay, interval),
  `core/countdown` (target date and time, time zone, what to show after), `core/progress` (label, value, max),
  `core/counter` (from, to, prefix, suffix), `core/modal` (trigger label, children).
- **Site:**
  - `core/navigation` and `core/navigation-link` (label, URL, children for submenus);
  - `core/table-of-contents` (levels to include), built from the page's heading blocks as described below;
  - `core/breadcrumbs` (from a new host data source);
  - `core/post-list` is **not a block type** but a template the Site definitions package registers: a `core/repeat`
    over a collection the host registers, such as recent posts, with editable children. *My call, from the builder
    check:* it needs no new contract and stays editable;
  - `core/search` (form posting to a URL the host sets).

### Headings and the table of contents

- **`core/heading` has an `anchor` setting** from its first version: plain text, empty by default. When empty, the
  output derives one from the heading's text. Stored from the start so it never needs a migration.
- **A behaviour cannot see the document,** so foundations add one contract: `DocumentStepInterface`, a step that
  sees the whole document once before resolving and adds values to the root context. *My call, from the builder
  check.* The Site definitions package registers one that lists the page's headings (level, text, anchor) under
  `core/headings`, and `core/table-of-contents` uses that context key.
- **Headings in included documents are not listed,** because they are resolved as separate documents. *My call.*

Every block gets an example, a catalogue entry and a description. Settings use the style vocabulary for appearance
rather than their own colour or spacing settings, except where a setting is the block's meaning, such as a
button's variant.

## Editor features

The reference editor gains, in this order:

1. Typing on the page, with a small formatting bar (bold, italic, link, code) over the selection.
2. Media picking and uploading for media settings (`x-control: "media"`).
3. Copy, cut and paste of blocks, including between documents (through the existing `insertBlocks`).
4. Inserting and saving templates, using the template library contracts that already exist.
5. Binding a setting to a data source from the inspector.
6. A global styles screen for design tokens and breakpoints, which needs those registries to be storable: they gain
   repository-backed implementations alongside the in-memory ones.
7. Multi-select, transforming a block into another type where the settings map, and locking a block. How
   transforms are declared and whether a lock is enforced by the server are decided in that part's plan, before
   anything changes the document format.

## Hosts

- **The demo host** binds the reference editor, places its page, serves package assets, and is how every part is
  checked by hand.
- **WordPress is out of scope,** as in the earlier design. The user chose the demo host only. `block-infra-wp` gains
  its media library implementation (part 3) but no admin screen and no editor wiring; `EditorInterface` is proven by
  tests and the demo.

## Phasing

Each part ends in working, tested software and gets its own plan.

1. **Foundations:**
   - the definitions and outputs split;
   - package titles and the per-plugin sections;
   - the editor seam and moving `BuilderAnnotations`;
   - formatted text and its sanitizer;
   - editable-setting marks;
   - the asset registry, the asset locator and output-declared assets;
   - the document step contract;
   - the contract suite's two modes;
   - splitting `block-structure`.
2. **Text** blocks, and typing on the page (editor feature 1).
3. **Media** contract, implementations and blocks, and media picking (editor feature 2).
4. **Actions** blocks.
5. **Interactive** blocks with their assets.
6. **Site** blocks.
7. **Editor features** 3 to 7.

## Left to the builder

- Exact method signatures, value object internals and file layout inside each package.
- The default outputs' exact markup and class names, provided they are accessible, escape every value and use the
  shared style vocabulary.
- The icon set's contents, the embed provider allow-list, and the social services list, provided each is a fixed
  list in the outputs or definitions package.
- How the reference editor's in-page editing is implemented, provided it sends ordinary `updateSettings`.

## Deferred from part 1

One critical and five important review findings were fixed: a cleaned link could carry an event attribute;
formatting was lost on Windows line endings and legacy entities; decorator and map settings were not cleaned when
stored; a bound list could put markup into formatted text; the outputs suite mode could pass a package that draws
nothing; and loading only the outputs half of a split failed silently (a package may no longer draw a type nobody
defined). Left:

- **Same-site checks** accept `/\` paths, which browsers read as another site, in formatted-text links and in
  `Asset::url`.
- **`Asset::file`** may name another package's files.
- **`outputOwner()`** goes stale when a host sets an output straight on the registry.
- **Sanitizer side effects:** the parser's wrapper can leak as text after raw-text elements, CDATA leaves `]]>`,
  relative links are unlinked, and `b` and `i` lose their formatting.
- **The binding escape** is HTML-specific and done at resolve time for every target.
- **The package suite** allows any `speedyspec/*-domain` dependency but refuses a third party's definitions package,
  and the definitions mode does not check that no outputs are registered.
- **The reference editor** declares only its entry script; its provider skips the asset when registered before the
  application provider; the demo's locator returns an empty URL for other packages.
- **Style:** one 121-character line, JavaScript indentation, `use` order, a title's wording, and public constants
  moved without a note.

## Part 2

The Text area shipped as `block-text-domain` (definitions) and `block-text` (outputs), and the reference editor
types into text on the page. Escape keeps what was typed rather than discarding it, since undo already takes a
change back. Duplicate heading anchors across one page are a known limit until the table of contents (part 6).

## Deferred from part 2

One critical and five important review findings were fixed: typing into a new table's cell replaced the table;
Backspace in an empty cell, caption or citation removed the whole block; Enter in a cell or caption added a new
block; a slow save could undo the cell typed before it; the link prompt could end typing; and the server kept `/\`
links (the part 1 item above). Left:

- **Tidying typed text:** a Chrome blank line is stored as two breaks; the contents of pasted `style` and `script`
  are kept as text; an attribute containing `>` leaves stray text; underline and non-breaking spaces are dropped.
- **Links:** a relative link such as `page.html` becomes `https://page.html`; a refused link gives no message.
- **Code** ending in a newline changes just by clicking in and out.
- **Heading ids** keep only ASCII letters, so headings in other scripts all get `heading`.
- **Style:** long lines in the text packages' tests.

## Part 3

The media contract shipped in `block-domain` with memory, filesystem and WordPress libraries; uploads are checked
by a `MediaUploadPolicyInterface`, also in `block-domain`, so the builder API does not depend on `block-app`. The
Media area shipped as `block-media-domain` and `block-media`, and the reference editor picks and uploads media.

- **Not set up is not broken.** A media block with nothing chosen yet raises no issue: visitors see nothing and
  editors a prompt. Only a chosen file that has gone or is the wrong kind is unavailable with an issue. Part of the
  reason is that the package suite requires every example to resolve without issues, and definitions cannot know a
  host's media ids.
- **Embeds and maps** show an error inside the reference editor, whose canvas runs no script; frames inside it
  inherit that. Visitors' pages are unaffected.

## Deferred from part 3

Five important review findings were fixed: ISO media files of any brand were taken as MP4 video; WordPress kept
an upload's typed extension; WordPress video and audio uploads failed making metadata; quick gallery picks were
lost; and the picker's search lost focus while typing. Left:

- **Picker:** an upload that finishes after its picker was swapped for another lands in the other setting; any
  failed lookup is shown as "No longer in the library"; search answers can arrive out of order.
- **Uploads in the demo:** the body is read before the size check, past PHP's own limit it still stores the file,
  and it has no request-forgery check.
- **Images** whose header gives 0×0 are drawn with no size.
- **WordPress:** about three queries per listed item.
- **Unused:** `ApiRequest` headers are not read yet.
- **Maps** build their OpenStreetMap address directly rather than through the embed list.
- **Style:** long lines in the demo server.

## Part 4

The Actions area shipped as `block-actions-domain` and `block-actions`, with one stylesheet asset, `core/actions`.

- **The canvas shows the page's stylesheets.** The builder API's render and preview answers gain `styles`, and the reference editor places them on its canvas. Scripts are never sent.
- **Buttons and social links follow part 3's "not set up" rule.** Without a usable link, visitors see nothing and editors see the block greyed out. A social link to another site is an `unavailable-media` issue.
- **Social links are drawn by name, not logo.** The logos are trademarks, so a plugin that wants them ships its own outputs package.
- **Scheme-relative button links are allowed.** A button may link to `//other.site`, which is no worse than an https link. Only formatted text keeps links on the site.

## Deferred from part 4

Two review findings were fixed:

- the editor's "Add a link" note was unreadable inside a button;
- common profile addresses were refused, such as `youtu.be`, `m.facebook.com` and `mobile.twitter.com`.

Left:

- **The list control:**
  - its buttons lose keyboard focus on redraw and are not numbered for screen readers;
  - Add item does not focus the new box and stays enabled at the limit;
  - its label names no element.
- **Canvas stylesheets** are placed links first, not strictly in dependency order.
- **Messages:** an http link for Website, RSS or Mastodon says the link is not to that service, rather than asking for https.
- **Empty markup for visitors:**
  - a cleared call-to-action heading is drawn as an empty heading;
  - an empty testimonial is drawn as an empty quote.
- **Editor only:**
  - `aria-disabled` on plain spans is ignored by assistive technology;
  - `renderRequest()` lost its docblock;
  - some demo lines are over 120 characters.

## Part 5

The Interactive area shipped as `block-interactive-domain` and `block-interactive`, with an inline stylesheet and a
module script that is a file in the package.

- **No script is needed.**
  - Accordions are `details` elements, grouped by `name` when only one may be open.
  - The modal uses the browser's `popover`. *My call:* that is not a true modal, since the rest of the page is not made inert, but it opens, closes on Escape and returns focus with no script.
  - Tabs show every panel in order, and slides scroll and snap.
- **What the script adds:** keyboard tabs, slider controls, a live countdown and a counting animation. It never writes markup from the page's text.
- **The demo serves package files** through `PackageFiles`, which is also its asset locator. This closes part 1's note that the demo's locator gave other packages an empty address. A `/view` page shows the home page as a visitor does, under a Content Security Policy.
- **Countdowns count to one moment.**
  - A local time the clocks skip becomes the next valid moment.
  - A repeated one counts to the earlier of the two. PHP picks the later, so the behaviour chooses the earlier itself.
  - A date the calendar does not have is reported, not rolled over into the next month.

## Deferred from part 5

Two review findings were fixed:

- element ids could clash between blocks whose ids differ only in punctuation;
- a cleared slider or tab label left a control with no name.

Left:

- **Ids across documents:** a host that draws several documents with the same block ids on one page can still repeat element ids.
- **The script:**
  - it re-scans the page on every change;
  - elements whose children arrive later are never set up;
  - tabs ignore a link into a later tab;
  - the counter jumps back to its start before counting;
  - the slider's Pause button comes after the slides;
  - very large counter values are clamped.
- **Countdown:**
  - its words are English and not pluralised;
  - an ended countdown keeps its "Starts in" label;
  - ambiguous zone names such as `GMT+5` are accepted.
- **No-script Safari:** keyboard users cannot scroll a slide row with nothing focusable in it.
- **Demo:**
  - `PackageFiles` serves any JS or CSS in a package, not only its assets;
  - some seed lines are over 120 characters.

## Part 6

The Site area shipped as `block-site-domain` and `block-site`.

- **The site map contract.** `SiteMapInterface` in `block-domain` is the "host data source" for breadcrumbs, the current menu link and search. Memory and WordPress versions are included.
- **Heading anchors are unique on the page** when the site package is loaded, which closes part 2's known limit:
  - the rules live in `block-domain`'s `HeadingAnchor`;
  - the site package's page step makes the ids unique;
  - the text package's heading reads its id from the step.
- **Submenus open with CSS alone.** A closed one is moved out of view, not hidden, because a hidden submenu's links cannot take keyboard focus, so `:focus-within` could never open it. The hand check found this.
- **The post list is a template.** It names the blocks it uses without depending on their packages.

## Deferred from part 6

One critical and five important review findings were fixed:

- **Hidden headings were listed.** The table of contents showed headings that visibility rules hide, and headings that could not be drawn. Any heading under visibility rules is now left out.
- **Included documents saw only themselves.** A menu or breadcrumbs in an included document showed the included document, not the page, and its headings repeated the page's ids.
- **Bound and empty headings gave empty links.**
- **Too many links were marked current.** Under query-named addresses every menu link was current, and in-page links were always current.
- **WordPress titles showed raw entities** in breadcrumbs.

Left:

- **Headings inside a repeat** are not made unique against the page.
- **Right-to-left pages** may scroll sideways because of the off-screen submenus.
- **The post list** draws an empty paragraph for a post with no excerpt.
- **Empty and non-Latin headings** get anchors that change when headings are reordered.


## Part 7

Editor features 3 to 7 shipped in the reference editor. The two questions this design left open were settled:

- **Transforms are declared as data on the type being turned.** `BlockType::transforms` lists `BlockTransform`s, each mapping a setting of the new type to one of its own. The server's `transform` operation carries it out, so every builder gets the same result. The block keeps its id, place and children; mapped settings and their bindings carry over; decorator settings stay where the decorator applies to the new type. A transform that would leave children where the new type cannot hold them is refused, never lossy. Paragraph, heading and quote turn into one another.
- **Locks are enforced by the server and stored as the `core/lock` extension**, `{move, remove}`, so the document format did not change. Moving or removing a locked block is refused, and removing a block checks everything inside it. `restore` is never checked, because undo is built on it: undoing the paste of a locked block must work. Anyone who may edit may unlock. The cost is that a client calling `restore` directly can get past a lock.

Also:

- **Stored site styles.** `StoredDesignTokenRegistry` and `StoredBreakpointRegistry` read their defaults until the first change, then only the record store, so removing every breakpoint stays removed. The style routes need the new `manage-styles` ability.
- **Breaking additions at 0.1.0:** `DocumentEditorInterface::removeBlocks()` and `transform()`, and `DataSourceRegistryInterface::all()`.
- **Clipboard text** is `{"type": "speedyspec/blocks", "format": 1, "blocks": [...]}`, with private settings kept, since only editors can copy.

## Deferred from part 7

Two important review findings were fixed:

- **Pasting copied blocks while typing** put their clipboard text into the paragraph. Blocks are now pasted as blocks after the paragraph, and pasting them into an inspector field is refused with a note.
- **A Shift-click on the page** could extend the browser's text selection, so copy took that text instead of the blocks. The editor now stops it. This could not be reproduced in the browser used for the hand check, so the fix is untested in Safari and Firefox.

Left:

- **Remove and Cut stay offered** for a block that has a locked block inside it. The server refuses, but its message names the inner block as "This block is locked against removal."
- **A failed template load** leaves "Loading templates…" up, with no way to try again.
- **Re-connecting the editor element** doubles its keyboard and clipboard listeners, so one paste pastes twice.
- **A refused site style change** puts the old value back in its field, so the person has to type it again.
- **One refusal message can be lost** from a many-block removal when a block id is a number. The edit is still refused.
- **Lock all and Unlock all are sent as a restore**, so an unrelated invalid block on the page refuses them, as it refuses undo.

All six were fixed afterwards, on `fix/block-library-part-7-minors`:

- Remove and Cut are turned off when any block inside is locked. The button says "Remove (a block inside is locked)", and the server's refusal says "A block inside this one is locked against removal."
- A failed template load shows "Templates could not be loaded." with Try again.
- Disconnecting the element takes its keyboard and clipboard listeners off.
- A refused site-style value stays in its field beside the reason.
- `removeBlocks` keeps unknown-id and lock refusals apart, so a numeric id loses nothing.
- Lock all and Unlock all use a new `lockBlocks` operation, added to `DocumentEditorInterface`. It changes only the listed blocks' locks, so nothing else on the page is checked again. The JS `withLock` helper was removed.
