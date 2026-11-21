# Block library part 7: editor features Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Give the reference editor the last five features of the design, editor features 3 to 7:
- copy, cut and paste of blocks, including between documents;
- inserting and saving templates;
- binding a setting to a data source from the inspector;
- a site styles screen for design tokens and breakpoints, which are now storable;
- selecting several blocks, turning a block into another type, and locking a block.

**Architecture:**
- **Most of this is editor work over the existing API.** Pasting uses `insertBlocks`, templates use `/templates` and `insertTemplate`, bindings use `setBinding` and `clearBinding`. The editor gains small, tested logic modules (`clipboard.js`, `templates.js`, `bindings.js`, `site-styles.js`, `selection.js`), and `builder-element.js` only wires them to the screen.
- **The server gains only what the design calls for:**
  - the list of data sources, so the inspector can offer them;
  - stored token and breakpoint registries, and routes to change them;
  - transforms declared on block types, and a `transform` operation;
  - a lock, enforced by the document editor.
- **The two decisions the design left to this plan:**
  - **Transforms are declared as data on the source block type:** `transforms: [{to: 'core/heading', settings: {text: 'text'}}]`, which maps each target setting to a source setting. The server does the transform, so every editor gets the same result. The editor only lists what the definition declares, keeping "anything an editor needs to know about a block is declared in its definition, as data".
  - **A lock is enforced by the server and stored as an extension, `core/lock`, with `move` and `remove` flags.** Extensions are already part of the document format, so the format does not change. `DocumentsPackage` registers `core/lock` as a decorator, so the inspector draws its settings like any other. The document editor refuses moving or removing a locked block. `restore` is not checked, because it is how undo works: undoing the paste of a locked block must be able to take it away again. Anyone who may edit may unlock: the lock prevents accidents, not access.

**Tech Stack:** PHP 8.4, Pest, Composer path repositories; plain ES modules and `node --test`.

**Spec:** `docs/superpowers/specs/2026-11-18-block-library-design.md` (in `wordpress-develop`). The relevant sections:
- "Editor features" items 3 to 7;
- "The editor seam" (what an editor must handle is defined by the API and the definitions, as data);
- "Anti-goals" (no editor code in block packages);
- "Phasing" item 7.

**Repository:** `wordpress-develop/packages/speedyspec-api-block`. Cut branch `feat/block-library-part-7` from `origin/main` (part 6 is merged as `22a0054`). Paths are relative to the repository's `packages/` directory.

## Global Constraints

- **Earlier constraints still hold.** Everything in parts 1 to 6's Global Constraints applies:
  - `declare(strict_types=1)`; 4 spaces;
  - braces of classes, methods and control structures on their own line;
  - docblocks per `wordpress-develop/.claude/rules/coding-style.md`, with `@since 0.1.0`; PHP lines at most 120 characters;
  - one commit per task with explicit paths, ending `Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>`.
- **Dependencies do not change.** New PHP goes in `block-domain` (contracts and value objects), `block-app` (implementations), `block-presentation` (routes) and `block-text-domain` (transform declarations). `block-builder`'s `composer.json` still requires only `speedyspec/block-domain`. Nothing depends on WordPress.
- **No editor code in block packages.** A definitions package declares transforms as data only.
- **Breaking interface changes are allowed at 0.1.0**, as in part 1. This plan adds `DataSourceRegistryInterface::all()`, and `DocumentEditorInterface::removeBlocks()` and `transform()`. Every implementation in the repository, test doubles included, is updated in the same task.
- **Names fixed by this plan:**
  - abilities: `BuilderAbility::ManageStyles` (`manage-styles`);
  - refusal kinds: `EditRefusalKind::NoTransform` (`no-transform`) and `EditRefusalKind::Locked` (`locked`);
  - the lock extension: `core/lock`, with boolean settings `move` and `remove`;
  - the saved-template prefix: `saved/`;
  - the clipboard text: a JSON object `{"type": "speedyspec/blocks", "format": 1, "blocks": [...]}`;
  - record store collections: `design-tokens`, `breakpoints`, and `style-registries` for the "has been changed" markers.
- **The editor stays usable without each feature.** When the server lacks what a feature needs, the feature is hidden rather than broken:
  - no `sources` in the vocabulary means no Bind button;
  - `manage: false` from `/templates` means no Save or Remove for templates;
  - `manageStyles: false` means no Site styles button;
  - no `core/lock` decorator means no lock settings.
- **What is not undoable.** Saving or removing a template and changing site styles are site changes, not document edits, so they are not in the document's undo history.
- **Plain words.** Interface text says what happens ("Copy", "Paste after", "Turn into", "Save as template", "Prevent moving"), in sentence case, with no jargon.

## Review Focus

1. **Clipboard text that is not blocks.**
   - Cases:
     - plain words;
     - malformed JSON;
     - JSON from another tool that has a `blocks` key but not our `type`;
     - our type with `format: 2`;
     - an empty block list;
     - a block without a string `type`.
   - Expected: each is ignored, the browser's own paste goes ahead, and no request is sent.
   - Test: `clipboard.test.js` (Task 1).
2. **Locks through undo and many-block edits.**
   - Cases:
     - lock a block, then undo back past the lock;
     - paste a locked block (or insert a template holding one), then undo the paste;
     - remove several blocks when one of them is locked;
     - remove the parent of a locked block.
   - Expected:
     - both undos work, since `restore` does not check locks;
     - a many-block removal with one locked block is refused as a whole, and nothing is removed;
     - removing the parent of a locked block is refused.
   - Tests: `DocumentEditorTest` (Task 7) and `store.test.js` (Task 8).
3. **Removing a breakpoint or token that stored styles still use.**
   - Expected: the page still draws without an exception, values at that width are left out, and a reference to the token falls back the way an unset custom property does.
   - Test: `StoredStyleRegistriesTest` (Task 4).
4. **Stored registries before and after the first change.**
   - Cases:
     - a store that has never been written to;
     - removing every breakpoint and reading again.
   - Expected:
     - before the first change, the registry shows the defaults it was given;
     - after every breakpoint is removed, a later read is still empty and does not show the defaults again.
   - Test: `StoredStyleRegistriesTest` (Task 4).
5. **Transforms that cannot keep everything.**
   - Cases:
     - a block with children turned into a type that holds none;
     - a block with a binding on a mapped setting;
     - a binding on a setting the target lacks;
     - a decorator that does not apply to the target.
   - Expected:
     - the first is refused and nothing changes;
     - the binding moves to the target setting;
     - the binding on a setting the target lacks is dropped;
     - the decorator's extension is dropped and other extensions are kept;
     - the block keeps its id.
   - Test: `DocumentEditorTest` (Task 6).

---

### Task 1: Copy, cut and paste

**Files:**
- Create: `block-builder/src/clipboard.js`, `block-builder/src/selection.js`, tests `block-builder/tests/clipboard.test.js` and `block-builder/tests/selection.test.js`.
- Modify:
  - `block-domain/src/Contracts/DocumentEditorInterface.php` (`removeBlocks()`);
  - `block-app/src/Building/DocumentEditor.php`, test `block-app/tests/Building/DocumentEditorTest.php`, and any test double implementing the interface;
  - `block-presentation/src/BuilderApi.php` (operation `removeBlocks`), test `block-presentation/tests/BuilderApiTest.php`;
  - `block-builder/src/store.js`, `block-builder/src/builder-element.js`, test `block-builder/tests/store.test.js`.

**Interfaces:**
- Produces:
  - **`DocumentEditorInterface::removeBlocks(Document $document, array $blockIds): EditResult`,** with `$blockIds` documented `list<string>`:
    - it removes every listed block, with its children, in one edit;
    - an id inside another listed block is simply removed with it;
    - any unknown id refuses the whole edit with `UnknownBlock`, as does an empty list;
    - `affected` is as `remove` gives it.
  - **API.** The operation is `{operation: 'removeBlocks', ids: [...]}`; `ids` must be a list of non-empty strings, else 400.
  - **`clipboard.js`:**
    - `clipboardText(blocks: Array<object>): string` writes the fixed clipboard JSON with the blocks as the document export gives them.
    - `readClipboard(text: string): ?Array<object>` gives the blocks back. It gives null for anything that is not that exact shape: `type` `speedyspec/blocks`, `format` 1, and a non-empty `blocks` list whose entries are objects with a string `type`.
    - `pastePosition(document, ids: string[]): {parent: ?string, index: number}` is right after the last of `ids` in document order. With no ids, or none still in the document, it is the end of the page.
  - **`selection.js`** (Task 8 adds more):
    - `pick(document, ids: string[]): Array<object>` gives the blocks with those ids in document order. Any id inside another picked block is left out, since its ancestor already carries it.
  - **Store:**
    - `paste(blocks, position)` sends `{operation: 'insertBlocks', blocks, position}` and selects the first block it added.
    - `removeMany(ids)` sends `remove` for one id, and one `removeBlocks` for more, so it is one undo step.
  - **Editor:**
    - **Keys.** It listens for `copy`, `cut` and `paste` on the element and on the canvas document. It ignores them while typing on the page or in any `input`, `textarea`, `select` or `[contenteditable]`, and when nothing is selected (copy and cut) or the text is not blocks (paste). It calls `preventDefault()` only when it acts.
    - **Cut** copies, then calls `removeMany`.
    - **A More button (⋯) on the block bar** opens a short list of actions: Copy, Cut, and Paste after. Paste after is shown only when `navigator.clipboard?.readText` exists. These use `navigator.clipboard.writeText` and `readText`. Escape closes the list and returns focus to the More button. Tasks 2 and 6 add to the list.
    - **Messages.** Copy says "Copied 1 block" (or "Copied N blocks") in the toast. A paste the server refuses shows its refusals as any refused edit does.

- [ ] **Step 1: Write the failing tests.**
  - `clipboard.test.js`:
    - `readClipboard(clipboardText([b]))` deep-equals `[b]`;
    - each case of Review Focus 1 gives null;
    - `pastePosition` after a nested block is `{parent: 'a', index: 2}` in the store test's `home()` shape when given `['a1']`, and `{parent: null, index: 2}` for `[]` and for `['gone']`.
  - `selection.test.js`: `pick(home, ['b', 'a1', 'a'])` gives `a` then `b`, with no separate `a1`.
  - `DocumentEditorTest`:
    - `removeBlocks` of `a1` and `b` leaves `a` with `a2`;
    - listing `a` and `a1` removes `a` and its children;
    - an unknown id, or an empty list, refuses and changes nothing.
  - `BuilderApiTest`: the `removeBlocks` operation, and a 400 for `ids` that are not a list of strings.
  - `store.test.js`:
    - `paste` sends `insertBlocks` and selects `affected[0]`;
    - `removeMany(['a1'])` sends `remove`;
    - `removeMany(['a1', 'b'])` sends one `removeBlocks`, and one undo restores both.
- [ ] **Step 2: Run** `cd block-app && vendor/bin/pest`, `cd block-presentation && vendor/bin/pest` and `cd block-builder && npm test`. Expected: FAIL.
- [ ] **Step 3: Implement** the operation, the modules, the store methods and the editor wiring.
- [ ] **Step 4: Run** them, plus `block-domain`. Expected: PASS.
- [ ] **Step 5: Commit** `feat(block-builder): copy, cut and paste blocks`.

---

### Task 2: Templates in the editor

**Files:**
- Create: `block-builder/src/templates.js`, test `block-builder/tests/templates.test.js`.
- Modify:
  - `block-presentation/src/BuilderApi.php` (`GET /templates` gains `manage`), test `block-presentation/tests/BuilderApiTest.php`;
  - `block-builder/src/api.js` (`templates()` returns `{libraries, manage}`), `block-builder/src/store.js`, `block-builder/src/builder-element.js`;
  - tests `block-builder/tests/api.test.js` and `store.test.js`.

**Interfaces:**
- Consumes: Task 1's More list.
- Produces:
  - **API.** `GET /templates` answers `{libraries, manage}`, where `manage` is whether the person has `BuilderAbility::ManageTemplates`.
  - **`templates.js`:**
    - `templateName(title: string): ?string` gives `saved/` plus a slug:
      - lower case, accents removed (NFKD, then combining marks dropped);
      - every run of other characters becomes `-`, with no `-` at either end;
      - `t-` in front when the slug does not start with a letter;
      - at most 64 characters, with no trailing `-` after cutting.
      - It gives null when nothing is left.
    - `matchTemplates(libraries, query: string): Array<{name, title, templates}>` keeps the templates whose title, description or category, or their library's title, contain the query, ignoring case. It drops libraries with no match, and keeps everything for an empty query.
    - `SAVED_LIBRARY = 'core/saved'`.
  - **Store.** State gains `templates: null | {libraries, manage}`. These are not undoable, and a failure sets `error`:
    - `loadTemplates()`;
    - `saveTemplate({block, title, description})` sends `client.saveTemplate({document: documentId, block, name: templateName(title), title, description})` and then reloads;
    - `removeTemplate(name)` sends it and then reloads.
  - **Editor:**
    - **The Add drawer gains two tabs, Blocks and Templates.** Templates loads on first open. It shows a section per library with the drawer's search, and inserts the chosen template at the drawer's current target through `insertTemplate`.
    - **Removing.** Saved templates have a Remove button when `manage` is true. It asks in place: the button becomes "Remove for good?" until clicked again or focus leaves it.
    - **Saving.** "Save as template" in the More list opens a form in the inspector:
      - Title, required;
      - Description;
      - the name it will be saved as;
      - "Replaces the saved template of this name." when that name is in the saved library;
      - Save and Cancel.
    - On success the toast says "Saved as a template".
    - With `manage` false, neither Save as template nor Remove is shown.

- [ ] **Step 1: Write the failing tests.**
  - `BuilderApiTest`: `manage` is true with `manage-templates` and false without it.
  - `templates.test.js`:
    - `templateName('Hero — Café 2')` is `saved/hero-cafe-2`;
    - `templateName('2 columns')` is `saved/t-2-columns`;
    - `templateName('!!!')` is null;
    - a 100-character title gives at most 70 characters in all (`saved/` plus 64);
    - `matchTemplates` by title, by library title, empty query, no match.
  - `api.test.js`: `templates()` returns `{libraries, manage}`.
  - `store.test.js`:
    - `saveTemplate` sends the derived name and the document id, then reloads the templates;
    - neither it nor `removeTemplate` changes `canUndo`.
- [ ] **Step 2: Run** `cd block-presentation && vendor/bin/pest` and `cd block-builder && npm test`. Expected: FAIL.
- [ ] **Step 3: Implement.**
- [ ] **Step 4: Run** both. Expected: PASS.
- [ ] **Step 5: Commit** `feat(block-builder): insert and save templates`.

---

### Task 3: Binding settings from the inspector

**Files:**
- Create: `block-builder/src/bindings.js`, test `block-builder/tests/bindings.test.js`.
- Modify:
  - `block-domain/src/Contracts/DataSourceRegistryInterface.php` (`all()`);
  - `block-app/src/Registries/DataSourceRegistry.php`, test `block-app/tests/Registries/DataSourceRegistryTest.php`, and any test double implementing the interface;
  - `block-presentation/src/BuilderApi.php` (vocabulary `sources`), test `BuilderApiTest.php`;
  - `block-builder/src/store.js`, `block-builder/src/builder-element.js`, test `store.test.js`;
  - `block-builder/demo/server.php` (a demo data source and a bound paragraph in the seed).

**Interfaces:**
- Produces:
  - **`DataSourceRegistryInterface::all(): array`,** documented `list<DataSourceInterface>`, sorted by name.
  - **The vocabulary gains `sources`:** a list of `{name, arguments, value}`, where `arguments` is `argumentsSchema()` and `value` is `valueSchema()`.
  - **`bindings.js`:**
    - `bindable(schema): string[]` names the top-level settings that may be bound: every property not marked `x-private`.
    - `bindingOf(block, setting): ?{source, arguments, fallback?}`.
    - `describeBinding(binding, sources): string` gives "From {source}", with the arguments after it as `name: value` pairs, comma-separated in brackets. It gives "From a source this site no longer has ({source})" when the source is not listed.
    - `bindingFrom(draft): object` builds the binding the API takes from the editor's form state, `{source, arguments, useFallback, fallback}`. It leaves out `fallback` unless `useFallback` is set, and leaves out arguments that are empty strings.
  - **Store:**
    - `setBinding(id, path, binding)` sends `{operation: 'setBinding', block: id, path, binding}`;
    - `clearBinding(id, path)` sends `{operation: 'clearBinding', block: id, path}`.
  - **Editor, Block tab:**
    - **Bind button.** Each bindable setting of the block's own settings has a "Bind" button by its label, shown only when the vocabulary lists sources.
    - **A bound setting** shows `describeBinding` in place of its control, with Change and Unbind buttons.
    - **The binding form,** shown inline under the label:
      - a Source select;
      - the source's arguments, drawn with `fields()` and the usual controls, so an unknown `x-control` falls back to a plain field;
      - "Show the current value when the source has none", which sets the fallback to the setting's current value;
      - Save and Cancel.
    - **Refusals** for that setting show under it, as for any setting.
    - **No typing into bound text.** A marked element whose setting is bound is not made editable on the page. A click on it selects the block and says "This text comes from {source}. Change it in the inspector."
  - **Demo.** It registers `ArrayDataSource('demo/site', ['name' => 'SpeedySpec demo', 'tagline' => 'Blocks for any editor'], ['type' => 'string'])`. The seed's first paragraph after the title is bound to `{source: 'demo/site', arguments: {key: 'tagline'}}`.

- [ ] **Step 1: Write the failing tests.**
  - `DataSourceRegistryTest`: `all()` gives the sources sorted by name, and none after `remove`.
  - `BuilderApiTest`: the vocabulary lists a registered source with its two schemas.
  - `bindings.test.js`:
    - `bindable` leaves out an `x-private` setting;
    - `describeBinding` for a known source and for a gone one;
    - `bindingFrom` with and without a fallback, and with an empty argument.
  - `store.test.js`: `setBinding` and `clearBinding` send their operations and are undoable.
- [ ] **Step 2: Run** the `block-app`, `block-presentation` and `block-builder` suites. Expected: FAIL.
- [ ] **Step 3: Implement.**
- [ ] **Step 4: Run** them, plus `block-domain`. Expected: PASS.
- [ ] **Step 5: Commit** `feat(block-builder): bind settings to data sources from the inspector`.

---

### Task 4: Stored tokens and breakpoints, and routes to change them

**Files:**
- Create: `block-app/src/Storage/StoredDesignTokenRegistry.php`, `block-app/src/Storage/StoredBreakpointRegistry.php`, test `block-app/tests/Storage/StoredStyleRegistriesTest.php`.
- Modify:
  - `block-domain/src/ValueObjects/BuilderAbility.php` (`ManageStyles`);
  - `block-app/src/Presentation/BreakpointRegistry.php` (`defaults()`) and `block-app/src/AppServiceProvider.php` (uses it);
  - `block-app/src/Storage/StorageServiceProvider.php`;
  - `block-presentation/src/BuilderApi.php`, test `block-presentation/tests/StylesTest.php`.

**Interfaces:**
- Produces:
  - **`BreakpointRegistry::defaults(): array`,** documented `list<Breakpoint>`: `tablet` at 768 and `desktop` at 1024, as now.
  - **`StoredDesignTokenRegistry(RecordStoreInterface $store, array $defaults = [], string $collection = 'design-tokens')`** implements `DesignTokenRegistryInterface`:
    - It checks names and values exactly as `DesignTokenRegistry::set()` does, with the same `InvalidStyleException` messages, and checks the defaults when constructed.
    - Until the first change, it reads the defaults.
    - The first change writes the defaults and a marker (collection `style-registries`, key = its own collection, value `1`), then applies itself. From then on, it reads only the store.
    - `all()` is sorted by name.
  - **`StoredBreakpointRegistry(RecordStoreInterface $store, ?array $defaults = null, string $collection = 'breakpoints')`** implements `BreakpointRegistryInterface` in the same way:
    - null defaults means `BreakpointRegistry::defaults()`;
    - each record is the width as a decimal string;
    - a stored value that is not a whole number of at least 1 raises `StorageException`, naming the breakpoint;
    - `all()` is ordered as `BreakpointRegistry::all()` orders.
  - **`StorageServiceProvider(array $tokens = [], ?array $breakpoints = null)`** also binds both registries over the record store, with those defaults.
  - **`BuilderAbility::ManageStyles = 'manage-styles'`.**
  - **Routes,** each needing `ManageStyles`:
    - `PUT /tokens/{name}` with `{value: string}` answers 200 `{tokens}`;
    - `DELETE /tokens/{name}` answers 200 `{tokens}`;
    - `PUT /breakpoints/{name}` with `{minWidth: integer}` answers 200 `{breakpoints}`;
    - `DELETE /breakpoints/{name}` answers 200 `{breakpoints}`;
    - `InvalidStyleException` becomes 422 `{error}`, and a missing or wrongly typed body field is a 400.
    - `tokens` and `breakpoints` are shaped as in the vocabulary.
  - **The vocabulary gains `manageStyles: bool`.**

- [ ] **Step 1: Write the failing tests.**
  - `StoredStyleRegistriesTest`, over `InMemoryRecordStore`:
    - defaults before any change;
    - a set survives a new registry over the same store;
    - after removing every breakpoint, a new registry is empty (Review Focus 4);
    - a bad name or value is refused before anything is written;
    - a corrupt width raises `StorageException`;
    - the provider binds both registries.
    - **Review Focus 3:** with a stored registry, remove `tablet` and the token `color.primary`, then render a block whose style extension has a `tablet` value and a `token:color.primary` colour. The render throws nothing, the CSS has no `768px` rule for it, and the base rule still has `var(--color-primary)`.
  - `StylesTest`:
    - each route's answer;
    - 403 without `manage-styles`;
    - 422 for `PUT /tokens/Bad` and for `minWidth: 0`;
    - 400 for a missing `value`;
    - `manageStyles` in the vocabulary.
- [ ] **Step 2: Run** `cd block-app && vendor/bin/pest` and `cd block-presentation && vendor/bin/pest`. Expected: FAIL.
- [ ] **Step 3: Implement.** If Review Focus 3's render throws, fix the cause in `StyleVocabulary` or `StyleCollector` in this task, and ledger it.
- [ ] **Step 4: Run** `block-domain`, `block-app`, `block-presentation`, `block-infra-memory`, `block-infra-filesystem` and `block-infra-pdo`, since the last three boot `StorageServiceProvider`. Expected: PASS.
- [ ] **Step 5: Commit** `feat(block-app): store design tokens and breakpoints, with routes to change them`.

---

### Task 5: The site styles screen

**Files:**
- Create: `block-builder/src/site-styles.js`, test `block-builder/tests/site-styles.test.js`.
- Modify:
  - `block-builder/src/api.js`, `block-builder/src/store.js`, `block-builder/src/builder-element.js`, tests `api.test.js` and `store.test.js`;
  - `block-builder/demo/server.php`: passes its two tokens to `StorageServiceProvider(tokens: …)` instead of setting them on every request, and allows `manage-styles`.

**Interfaces:**
- Consumes: Task 4's routes and `manageStyles`.
- Produces:
  - **`api.js`:** `setToken(name, value)`, `removeToken(name)`, `setBreakpoint(name, minWidth)` and `removeBreakpoint(name)`. Each encodes the name as one path segment and returns the answer's `tokens` or `breakpoints`.
  - **`site-styles.js`:**
    - `tokenGroups(tokens: object): Array<{title, tokens: Array<{name, value}>}>` groups by the part before the first `.`, with titles in words ("Color", "Space"). Names without a `.` go in "Other", last. Groups and tokens are sorted by name.
    - `isColourToken(name): boolean` is true for names that start with `color.`.
    - `tokenNameProblem(name): ?string` mirrors the server's pattern, `^[a-z][a-z0-9-]*(\.[a-z0-9-]+)*$`. It gives "Use lower-case letters, digits and hyphens, with dots between parts, such as color.accent." The server stays the authority.
    - `widthProblem(raw): ?string` gives "Use a whole number of pixels, 1 or more." for anything else.
  - **Store:** `setToken`, `removeToken`, `setBreakpoint` and `removeBreakpoint` each return `Promise<?string>`:
    - null on success, after putting the answer into `state.vocabulary` and refreshing the preview;
    - the server's message when it answers 422 or 403;
    - nothing here enters the undo history.
  - **Editor:**
    - **A "Site styles" button** in the top bar, shown when `vocabulary.manageStyles`, opens a `styles` drawer.
    - **Tokens,** a section per group:
      - each row has the name, a value field (a colour input beside the text for colour tokens) saved on change, and a Remove button;
      - an Add row takes a name and a value.
    - **Breakpoints:**
      - each row has the name, a width field and a Remove button;
      - an Add row takes a name and a width;
      - a note says "Styles set for a removed width stop applying."
    - **Problems** show under their row, from the local check first and then from the server.

- [ ] **Step 1: Write the failing tests.**
  - `site-styles.test.js`:
    - grouping and order with an undotted name;
    - `isColourToken`;
    - `tokenNameProblem` for `color.accent` (null), `Color`, `color..x` and `1x`;
    - `widthProblem` for `768`, `0`, `7.5` and `''`.
  - `api.test.js`: `setToken('color.primary', '#fff')` sends `PUT /tokens/color.primary` with `{value}`.
  - `store.test.js`:
    - a successful `setToken` updates `vocabulary.tokens` and re-renders the preview;
    - a 422 resolves to its message and leaves the vocabulary alone;
    - `canUndo` is unchanged.
- [ ] **Step 2: Run** `cd block-builder && npm test`. Expected: FAIL.
- [ ] **Step 3: Implement**, including the demo change.
- [ ] **Step 4: Run** `npm test`. Expected: PASS.
- [ ] **Step 5: Commit** `feat(block-builder): add the site styles screen`.

---

### Task 6: Turning a block into another type

**Files:**
- Create: `block-domain/src/ValueObjects/BlockTransform.php`, test `block-domain/tests/ValueObjects/BlockTransformTest.php`.
- Modify:
  - `block-domain/src/ValueObjects/BlockType.php` (`transforms`), `CatalogueEntry.php` (`transforms` in `toArray()`), `EditRefusalKind.php` (`NoTransform`), `Contracts/DocumentEditorInterface.php` (`transform()`);
  - `block-app/src/Building/DocumentEditor.php`, test `block-app/tests/Building/DocumentEditorTest.php`;
  - `block-presentation/src/BuilderApi.php` (operation `transform`), test `BuilderApiTest.php`;
  - `block-text-domain/src/TextTypesPackage.php`, test `block-text-domain/tests/TypesTest.php`;
  - `block-builder/src/store.js`, `block-builder/src/builder-element.js`, test `store.test.js`.

**Interfaces:**
- Produces:
  - **`BlockTransform(BlockName $to, array $settings)`:**
    - `$settings` maps a target setting to a source setting, both non-empty strings. Anything else raises `InvalidBlockTypeException`.
    - `fromArray(['to' => …, 'settings' => […]])` and `toArray()`.
  - **`BlockType`:**
    - It gains a last constructor parameter, `array $transforms = []`, documented `list<BlockTransform>`, read from `fromArray`'s `transforms`, a list of arrays.
    - The constructor refuses:
      - a non-`BlockTransform` entry;
      - two entries to one type;
      - a transform to itself;
      - a source setting its own schema does not declare.
    - `transformTo(BlockName $to): ?BlockTransform`.
  - **`CatalogueEntry::toArray()`** gains `transforms`, the target names.
  - **`EditRefusalKind::NoTransform = 'no-transform'`.**
  - **`DocumentEditorInterface::transform(Document $document, string $blockId, BlockName $to): EditResult`.** `DocumentEditor` refuses an unknown block, an unknown source or target type, or no declared transform (`NoTransform`). Otherwise it builds the replacement:
    - **Identity:** the same id, the target type at its current version.
    - **Settings:** each mapped source value the block has, cleaned with the target's schema as `insert` cleans.
    - **Bindings:** a binding whose first path segment is a mapped source moves to the target name. Other bindings are dropped.
    - **Extensions:** those whose decorator is registered and applies to the target are kept. Others are dropped.
    - **Children** are kept.
    - **Checks.** The replacement is checked as pasted blocks are (`vetted`: placement under its parent, settings, bindings and children). Any refusal refuses the whole edit.
    - `affected` is `[id]`.
  - **API.** The operation is `{operation: 'transform', block, type}`.
  - **Text definitions:**
    - `core/paragraph` declares transforms to `core/heading` and `core/quote`;
    - `core/heading` to `core/paragraph` and `core/quote`;
    - `core/quote` to `core/paragraph` and `core/heading`;
    - each maps `{text: 'text'}`.
  - **Store:** `transform(id, type)` sends the operation, and the selection stays on the same id.
  - **Editor.** "Turn into" in the More list shows the selected block's catalogue `transforms` that are in the catalogue, by title. It is left out when there are none.

- [ ] **Step 1: Write the failing tests.**
  - `BlockTransformTest`: a valid map, an empty target name, a non-string source, and the `fromArray`/`toArray` round trip.
  - `block-domain` `BlockTypeTest` (or the existing type tests): each constructor refusal, and `transformTo`.
  - `DocumentEditorTest`:
    - paragraph to heading keeps the id and text, and gets level 2;
    - quote to paragraph drops `citation`;
    - no declared transform gives `NoTransform`;
    - and Review Focus 5's cases.
  - `BuilderApiTest`: the `transform` operation, and catalogue entries listing `transforms`.
  - `TypesTest`: the six text transforms are declared.
  - `store.test.js`: `transform` sends the operation and keeps the selection.
- [ ] **Step 2: Run** the `block-domain`, `block-app`, `block-presentation`, `block-text-domain` and `block-builder` suites. Expected: FAIL.
- [ ] **Step 3: Implement.**
- [ ] **Step 4: Run** them, plus every definitions and outputs package's suite (the package contract suite builds `BlockType`s). Expected: PASS.
- [ ] **Step 5: Commit** `feat(block-app): turn a block into another type where its settings map`.

---

### Task 7: Locking a block

**Files:**
- Create: `block-domain/src/ValueObjects/BlockLock.php`, test `block-domain/tests/ValueObjects/BlockLockTest.php`.
- Modify:
  - `block-domain/src/ValueObjects/EditRefusalKind.php` (`Locked`);
  - `block-app/src/Documents/DocumentsPackage.php` (registers the decorator);
  - `block-app/src/Building/DocumentEditor.php`, test `DocumentEditorTest.php`;
  - `block-builder/src/inline.js` (`removableWhenEmpty`), `block-builder/src/builder-element.js`, test `block-builder/tests/inline.test.js`.

**Interfaces:**
- Produces:
  - **`BlockLock`:**
    - `NAMESPACE = 'core/lock'`;
    - readonly `bool $move` and `bool $remove`;
    - `of(Block $block): self` reads the extension, treating anything other than `true` as false;
    - `isLocked(): bool`;
    - `decorator(): Decorator` is the `core/lock` decorator for every type, with schema `{type: object, properties: {move: {type: boolean, default: false, title: 'Prevent moving'}, remove: {type: boolean, default: false, title: 'Prevent removing'}}}`.
  - **`EditRefusalKind::Locked = 'locked'`.**
  - **`DocumentsPackage`** registers `BlockLock::decorator()`.
  - **`DocumentEditor` refuses with `Locked`,** naming the locked block's id:
    - `move` of a block whose lock has `move`;
    - `remove` and `removeBlocks` when any block in a removed subtree has `remove`; `removeBlocks` is then refused as a whole.
    - It never checks `restore`, which undo and redo use.
    - It never refuses changing `core/lock` itself, so anyone who may edit may unlock.
    - The messages are "This block is locked in place." and "This block is locked against removal."
  - **`removableWhenEmpty(block, schema, path)`** is false for a block whose lock has `remove`.
  - **Editor:**
    - **Move lock:** Move up, Move down, the parent's indent and outdent, and dragging are turned off.
    - **Remove lock:** Remove and Cut are turned off.
    - **Outline:** a locked block's row shows a lock mark titled "Locked".
    - **Lock settings** appear in the Block tab as a "Lock" group, drawn generically like any other decorator, with no special code.

- [ ] **Step 1: Write the failing tests.**
  - `BlockLockTest`:
    - `of()` with no extension, `{move: true}` and `{remove: 'yes'}` (false);
    - the decorator's namespace and schema.
  - `DocumentEditorTest`:
    - each refusal above;
    - moving or removing an unlocked sibling still works;
    - unlocking through `updateExtension` is accepted;
    - **Review Focus 2:**
      - lock, then `restore` the blocks from before the lock, is accepted;
      - `insertBlocks` of a locked block, then `restore` of the blocks from before, is accepted;
      - `removeBlocks` listing a locked block and an unlocked one is refused, and the document is unchanged;
      - removing a locked block's parent is refused.
  - `inline.test.js`: a locked, empty paragraph is not removable.
- [ ] **Step 2: Run** the `block-domain`, `block-app` and `block-builder` suites. Expected: FAIL.
- [ ] **Step 3: Implement.**
- [ ] **Step 4: Run** them, plus `block-presentation`. Expected: PASS.
- [ ] **Step 5: Commit** `feat(block-app): lock blocks against moving and removal`.

---

### Task 8: Selecting several blocks, the demo, a hand check and documentation

**Files:**
- Modify:
  - `block-builder/src/selection.js`, `block-builder/src/store.js`, `block-builder/src/builder-element.js`, tests `selection.test.js` and `store.test.js`;
  - `block-builder/demo/server.php`;
  - repository `README.md` and `block-builder/README.md`;
  - `wordpress-develop/docs/superpowers/specs/2026-11-18-block-library-design.md` (a part 7 note).

**Interfaces:**
- Consumes: Tasks 1 to 7.
- Produces:
  - **`selection.js`:**
    - `toggleInSelection(document, selection: string[], id): string[]`:
      - it removes `id` if it is there;
      - it adds `id` when the selection is empty or `id` has the same parent as the selection's first block;
      - otherwise it starts again with `[id]`;
      - the result is in document order.
    - `withLock(blocks, ids, lock: {move: boolean, remove: boolean}): Array<object>` gives a copy with `extensions['core/lock']` set to `lock` on those ids, or removed when both flags are false.
  - **Store:**
    - State gains `selection: string[]`. It always includes `selected` when that is not null, and edits keep only the ids still in the document.
    - `select(id)` sets `selection` to `[id]`, or `[]` for null.
    - `toggle(id)` applies `toggleInSelection`, and sets `selected` to the id just added or else to the last left.
    - `lockMany(ids, lock)` sends one `restore` of `withLock(...)`, so it is one undo step, as `removeMany` is.
  - **Editor:**
    - **Selecting.** Shift-, Ctrl- or Cmd-click on a canvas block or an outline row calls `toggle`. Every selected block is marked on the canvas and in the outline, and outline rows say "selected" to screen readers. Escape clears the selection when not typing.
    - **With more than one block selected:**
      - the block bar is hidden;
      - the inspector shows "{N} blocks selected", with Copy, Cut, Remove, "Lock all" and "Unlock all";
      - Copy, Cut and paste use the whole selection through Task 1's `pick` and `pastePosition`;
      - Remove uses `removeMany`;
      - Lock all sets `{move: true, remove: true}`.
  - **Demo.** The seed gains a locked call-to-action (`core/lock` `{move: true, remove: true}`), so a fresh demo shows a lock. It allows `manage-styles` (Task 5).
  - **Documentation:**
    - the repository README gains an "Editor features" section, covering clipboard, templates, bindings, site styles, transforms, locks and multi-select. It names the `transform` operation, the `core/lock` extension, the style routes and `manage-styles`, and the clipboard format;
    - the builder README gains the same from the editor's side;
    - the design note records the two decisions above.

- [ ] **Step 1: Write the failing tests.**
  - `selection.test.js`:
    - `toggleInSelection`: adding a sibling, removing one, a non-sibling restarting, and document order;
    - `withLock`: setting and clearing.
  - `store.test.js`:
    - `toggle` builds the selection and moves `selected`;
    - removing a selected block by an edit drops it from `selection`;
    - `lockMany` sends one `restore`;
    - **Review Focus 2:** when the server refuses `removeMany`'s `removeBlocks`, its refusals are shown and the document is unchanged.
- [ ] **Step 2: Run** `cd block-builder && npm test`. Expected: FAIL.
- [ ] **Step 3: Implement**, with the demo and documentation.
- [ ] **Step 4: Run** every PHP suite and both `npm test`s (`block-builder`, `block-interactive`). Expected: PASS.
- [ ] **Step 5: Check by hand.** Delete the demo's `documents/` and the record store's `design-tokens`, `breakpoints` and `style-registries` collections first, so everything reseeds. In the editor (`/`):
  - **Clipboard:**
    - copy a block with Ctrl/Cmd+C and paste it after another block with Ctrl/Cmd+V;
    - cut a block, and undo the cut;
    - in a second tab, create a second document and paste the first's block into it;
    - pasting plain text while a block is selected does nothing.
  - **Templates:**
    - insert the post-list template from the Templates tab;
    - save a section as a template, insert it, then remove it.
  - **Bindings:**
    - the bound tagline paragraph shows "From demo/site (key: tagline)" and cannot be typed into;
    - unbind it, type into it, then bind a heading to `demo/site` `name`;
    - on `/view` the heading shows "SpeedySpec demo".
  - **Site styles:**
    - change `color.primary`, and the canvas updates;
    - reload, and the change has stayed;
    - add a breakpoint `wide` at 1280, and the Style tab offers "From wide";
    - remove it.
  - **Transforms:** turn a paragraph into a heading and back; undo restores the original.
  - **Locks:**
    - the locked call-to-action's move and remove buttons are off;
    - untick "Prevent removing", and Remove works.
  - **Multi-select:**
    - Shift-click three sibling blocks, then copy and paste them;
    - Lock all, then Unlock all;
    - remove all three, then undo.
- [ ] **Step 6: Commit** `feat(block-builder): select several blocks, and document the editor features`.

---

## Out of scope for this plan

- Moving several blocks at once by dragging or by the move buttons: dragging still moves one block.
- Transforms that change children, such as a list into paragraphs or an accordion into tabs.
- Binding settings inside lists or decorators. Only a block's own top-level settings are offered.
- Abilities for unlocking. Any editor may unlock.
- Site style changes in an undo history.
- Pasting blocks from other editors' formats, or turning pasted HTML into blocks.
