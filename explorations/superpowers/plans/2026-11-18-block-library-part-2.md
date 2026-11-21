# Block library part 2: text blocks and typing on the page Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add the text area of the block library — heading, paragraph, list and list item, quote, code, table, separator and spacer — as a definitions package and a default outputs package, and let a person type into text blocks directly on the page in the reference editor.

**Architecture:** `block-text-domain` registers the nine block types and their behaviours; `block-text` registers their outputs. Outputs print formatted text through `$input->escaper->richText()` and mark what can be typed into with `$input->editable()`, both from part 1. The reference editor makes marked elements editable inside its sandboxed frame, tidies what the browser produced, and sends an ordinary `updateSettings`. A small formatting bar works on the selection.

**Tech Stack:** PHP 8.4, Pest, Composer path repositories; plain ES modules and `node --test`.

**Spec:** `docs/superpowers/specs/2026-11-18-block-library-design.md` (in `wordpress-develop`): "The blocks" (Text), "Formatted text", "Editing on the page", "Headings and the table of contents", "Editor features" item 1.

**Repository:** `wordpress-develop/packages/speedyspec-api-block`; branch from `origin/main` (which has part 1 and its review fixes). Paths are relative to its `packages/` directory.

## Global Constraints

- Everything in part 1's Global Constraints holds: `declare(strict_types=1)`; 4 spaces; braces of classes, methods and control structures on their own line; docblocks per `wordpress-develop/.claude/rules/coding-style.md`; lines at most 120 characters; one commit per task with explicit paths, ending `Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>`.
- `block-text-domain` requires only `speedyspec/block-domain`. `block-text` requires `speedyspec/block-domain` and `speedyspec/block-text-domain`.
- Block names are `core/heading`, `core/paragraph`, `core/list`, `core/list-item`, `core/quote`, `core/code`, `core/table`, `core/separator`, `core/spacer`, with category `text`. Their settings are fixed by this plan: once released, changing them needs a migration.
- Every formatted-text setting is `{"type": "string", "default": "", "x-control": "rich-text"}` and is printed only through `$input->escaper->richText()`. Every plain-text setting is printed only through `$input->escaper->text()`.
- Appearance comes from the shared `core/style` settings through the attribute bag, which every output writes on its outermost element. No block has its own colour, size or spacing settings, except `core/spacer`'s height, which is its meaning.
- Outputs mark typeable settings with `$input->editable()` and never write editing marks themselves.
- The reference editor knows no block type by name. Everything it does for text comes from the `data-block-setting` marks and the settings schema.

## Review Focus

1. **Typed or pasted content that is not clean** (pasted HTML from a word processor, `<b>`/`<i>`/`<div>`/`<span style>`, `javascript:` links typed into the link field, a trailing `<br>` the browser adds). Expected: what reaches the server is close to the allowed subset, and the server's cleaning has the last word. Tasks 2 and 3.
2. **A heading's anchor** that is empty, has spaces or markup, or repeats another heading's. Expected: a valid `id` derived from the text, never markup in the attribute; repeated anchors are a known limit, recorded, not a crash. Task 2.
3. **A table whose stored rows are ragged, empty or not lists.** Expected: it draws every row it can, padding short rows, and never throws. Task 2.
4. **Editing a setting inside a list** (`rows.1.2` in a table). Expected: only that cell changes; the rest of the table is kept exactly. Task 3.
5. **Keys while typing** (Enter, Shift+Enter, Backspace at the start of an empty block, Escape, Ctrl/Cmd+Z). Expected: Enter adds a block of the same type after and moves the cursor there; Shift+Enter is a line break; Backspace in an empty block removes it; Escape stops editing; undo inside a field is the field's own. Task 3.

---

### Task 1: Text block definitions

**Files:**
- Create: `block-text-domain/` (`composer.json`, `phpunit.xml`, `src/TextTypesPackage.php`, `src/HeadingBehaviour.php`, `tests/Pest.php`, `tests/PackageContractTest.php`, `tests/TypesTest.php`)

**Interfaces:**
- Produces: `SpeedySpec\Block\Text\Domain\TextTypesPackage` (name `core/text-types`, title "Text", description "Headings, paragraphs, lists, quotes, code and tables.").
- Produces: these settings, each with `title`, an example, and the policies given:

| Type | Settings | Children / parents |
|---|---|---|
| `core/heading` | `level` integer 1–6, default 2; `text` formatted; `anchor` string, pattern `^([a-z][a-z0-9-]{0,63})?$`, default `''` | none |
| `core/paragraph` | `text` formatted | none |
| `core/list` | `ordered` boolean, default false; `start` integer ≥ 1, default 1 | `core/list-item` only |
| `core/list-item` | `text` formatted | `core/list` only (for nesting); parent `core/list` only |
| `core/quote` | `text` formatted; `citation` string, default `''` | none |
| `core/code` | `code` string, default `''`, `x-control: code`; `language` string, pattern `^([a-z0-9+#-]{0,32})$`, default `''` | none |
| `core/table` | `header` boolean, default true; `rows` array of arrays of formatted strings, default `[["", ""], ["", ""]]`, `x-control: table`; `caption` formatted | none |
| `core/separator` | none | none |
| `core/spacer` | `height` responsive length, pattern as `core/style` lengths, default `2rem`, `x-control: responsive` | none |

- Produces: `HeadingBehaviour` gives the view `level`, `text` and `id`: the stored anchor when set, otherwise one derived from the text (tags removed, entities decoded, lower-cased, runs of anything outside `a-z0-9` replaced with one hyphen, trimmed, at most 64 characters, prefixed `h-` when it would start with a digit, `heading` when empty).

- [ ] **Step 1: Write the failing tests**

```php
describe('TextTypesPackage', function () {
    blockPackageContract(fn () => new TextTypesPackage(), __DIR__ . '/../composer.json', [], 'definitions');
});
test('the text blocks have the settings this plan fixes', function () {
    expect(array_keys($types->get(BlockName::fromString('core/heading'))->settingsSchema['properties']))
        ->toBe(['level', 'text', 'anchor'])
        ->and($types->get(BlockName::fromString('core/table'))->settingsSchema['properties']['rows']['items']['items']['x-control'])
        ->toBe('rich-text');
});
test('a heading gets an id from its anchor, or from its text', function (array $settings, string $id) {
    expect(resolved(block('h', 'core/heading', $settings))->view->values['id'])->toBe($id);
})->with([
    [['anchor' => 'intro', 'text' => 'Whatever'], 'intro'],
    [['text' => 'Hello, <em>world</em> &amp; more'], 'hello-world-more'],
    [['text' => '2026 plans'], 'h-2026-plans'],
    [['text' => ''], 'heading'],
    [['text' => str_repeat('a', 100)], str_repeat('a', 64)],
]);
test('a list item goes only in a list, and a list may sit in a list item', function () {
    $decide = fn (string $child, ?string $parent) => $placement->decide(
        BlockName::fromString($child),
        $parent === null ? null : BlockName::fromString($parent),
    )->allowed;

    expect($decide('core/list-item', 'core/list'))->toBeTrue()
        ->and($decide('core/list', 'core/list-item'))->toBeTrue()
        ->and($decide('core/list-item', null))->toBeFalse()
        ->and($decide('core/paragraph', 'core/list'))->toBeFalse();
});
test('settings outside the rules are refused when edited', function (string $type, array $settings) {
    expect(refusals($editor->insert(doc(), BlockName::fromString($type), Position::root(0), $settings)))
        ->toContain(EditRefusalKind::InvalidSettings);
})->with([
    ['core/heading', ['level' => 7]],
    ['core/heading', ['anchor' => 'Has Spaces']],
    ['core/code', ['language' => '"><script>']],
    ['core/spacer', ['height' => '1px;} x{']],
    ['core/table', ['rows' => 'nope']],
]);
```

`tests/Pest.php` gives `block()`, `resolved()`, `editor()` and `refusals()` helpers over a container with the memory and application providers and this package loaded.

- [ ] **Step 2: Run** `cd block-text-domain && composer install && vendor/bin/pest --colors=never`. Expected: FAIL (class not found).
- [ ] **Step 3: Implement.**
- [ ] **Step 4: Run.** Expected: PASS.
- [ ] **Step 5: Commit** `feat(block-text): add the text block definitions`.

---

### Task 2: Text block outputs

**Files:**
- Create: `block-text/` (`composer.json`, `phpunit.xml`, `src/TextPackage.php`, one output class per block or a small set of shared ones, `tests/Pest.php`, `tests/PackageContractTest.php`, `tests/OutputTest.php`)

**Interfaces:**
- Consumes: `TextTypesPackage` (Task 1); `RenderInput::editable()`, `OutputEscaperInterface::richText()`, `StyleCollectorInterface` (part 1).
- Produces: `SpeedySpec\Block\Text\TextPackage` (name `core/text`, title "Text, drawn"), `TextPackage::from(BlockServiceContainer)`.
- Produces this markup, where `{attrs}` is the attribute bag (with `editable()` on the element holding each typeable setting):

| Type | Front markup |
|---|---|
| `core/heading` | `<h{level} id="{id}"{attrs}>{text}</h{level}>` |
| `core/paragraph` | `<p{attrs}>{text}</p>` |
| `core/list` | `<ul{attrs}>…</ul>`, or `<ol{attrs} start="{start}">` when ordered and `start` is not 1 |
| `core/list-item` | `<li{attrs}>{text}{children}</li>` |
| `core/quote` | `<blockquote{attrs}><p>{text}</p><cite>{citation}</cite></blockquote>`, no `cite` when empty |
| `core/code` | `<pre{attrs}><code class="language-{language}">{code}</code></pre>`, no class when empty |
| `core/table` | `<figure{attrs}><table><thead>…</thead><tbody>…</tbody></table><figcaption>{caption}</figcaption></figure>`; `th scope="col"` in the header row; no `thead` when `header` is false; no `figcaption` when empty |
| `core/separator` | `<hr{attrs}>` |
| `core/spacer` | `<div aria-hidden="true"{attrs}></div>` with its height as collected style rules |

- Rules: a table's columns are the longest row's length; shorter rows are padded with empty cells; a stored `rows` that is not a list of lists draws an empty table. An empty paragraph or heading still draws its element, so it can be clicked and typed into. Table cells are marked with the setting path `rows.{r}.{c}`; the caption with `caption`.

- [ ] **Step 1: Write the failing tests**

```php
describe('TextPackage', function () {
    blockPackageContract(
        fn () => TextPackage::from(textHost()),
        __DIR__ . '/../composer.json',
        [fn () => new TextTypesPackage()],
        'outputs',
    );
});
test('each text block draws its markup', function (Block $block, string $html) {
    expect(page($block))->toBe($html);
})->with([
    // one case per type, its settings chosen to show each rule in the markup table above
    [block('h', 'core/heading', ['level' => 3, 'text' => 'Hi']), '<h3 id="hi">Hi</h3>'],
    [block('l', 'core/list', ['ordered' => true, 'start' => 3], block('i', 'core/list-item', ['text' => 'a'])), '<ol start="3"><li>a</li></ol>'],
    [block('q', 'core/quote', ['text' => 'a', 'citation' => '']), '<blockquote><p>a</p></blockquote>'],
    [block('s', 'core/separator'), '<hr>'],
]);
test('formatted text is cleaned and plain text is escaped', function () {
    expect(page(block('p', 'core/paragraph', ['text' => '<em>a</em><script>x</script>'])))->toBe('<p><em>a</em>x</p>')
        ->and(page(block('c', 'core/code', ['code' => '<b>&</b>'])))->toBe('<pre><code>&lt;b&gt;&amp;&lt;/b&gt;</code></pre>');
});
test('a ragged or broken table draws what it can', function () {
    expect(page(block('t', 'core/table', ['rows' => [['a', 'b', 'c'], ['d']]])))->toContain('<td>d</td><td></td><td></td>')
        ->and(page(block('t', 'core/table', ['rows' => [['a'], 'x']])))->toContain('<table>');
});
test('while editing, typeable settings are marked and visitors never see the marks', function () {
    expect(page($table, editing: true))->toContain('data-block-setting="rows.1.0"')->toContain('data-block-setting="caption"')
        ->and(page($table))->not->toContain('data-block-setting');
});
test('a spacer\'s height is a style rule that may differ by width', function () {
    expect($css)->toContain('{height:2rem}')->toContain('@media (min-width:768px){.' . $class . '{height:4rem}}');
});
```

`textHost()` builds a container with the memory and application providers. `page(Block, bool $editing = false)` draws one block in a host with the memory and application providers, the presentation package and both text packages loaded; with `$editing`, in admin mode with `RenderRequest::editing` set.

- [ ] **Step 2: Run** `cd block-text && composer install && vendor/bin/pest --colors=never`. Expected: FAIL.
- [ ] **Step 3: Implement.**
- [ ] **Step 4: Run.** Expected: PASS.
- [ ] **Step 5: Commit** `feat(block-text): add the text block outputs`.

---

### Task 3: Typing on the page

**Files:**
- Create: `block-builder/src/inline.js`, `block-builder/tests/inline.test.js`
- Modify: `block-builder/src/forms.js` (`withValue` gains paths), `block-builder/src/builder-element.js`, `block-builder/src/styles.js`, `block-builder/src/store.js` (an `insertAfter` helper), `block-builder/tests/forms.test.js`, `block-builder/tests/store.test.js`

**Interfaces:**
- Produces in `src/inline.js` (no DOM; strings and plain objects in, strings out):
  - `tidyTyped(html: string): string` — turns what a browser's editing produced into the allowed subset as closely as a client can: `b`→`strong`, `i`→`em`, `strike`/`del`→`s`; `div` and `p` boundaries become `<br>`; `span`, `font` and any attribute other than `href` on `a` are removed with their text kept; one trailing `<br>` is dropped; `&nbsp;` becomes a space.
  - `keyAction({key, shiftKey, metaKey, ctrlKey, empty, atStart}): 'commit-split'|'line-break'|'remove'|'stop'|null`.
  - `safeLink(text: string): ?string` — the same allow-list as the server (http, https, mailto, tel, a path on the same site, `#`), adding `https://` to a bare domain such as `example.com`; null for anything else.
- Changes: `withValue(settings, path, value)` accepts a dotted path (`rows.1.2`) and copies only what lies along it; a plain name behaves as before.
- Produces: `store.insertAfter(id, type)` inserts a block of the given type directly after `id` and selects it.
- Changes in the element:
  - Clicking an element marked `data-block-setting` selects its block and makes the element editable (`contenteditable="true"`, or `plaintext-only` when the setting's schema is not `rich-text`). The frame still runs no script; the builder edits it from outside.
  - Leaving the element, or Enter, sends `updateSettings` with the tidied value at that path, only if it changed. Formatted text is tidied with `tidyTyped`; plain text is sent as text.
  - Enter (`commit-split`) also inserts a block of the same type after and puts the cursor in it when the page is redrawn. Shift+Enter inserts a line break. Backspace in an empty element (`remove`) removes the block and selects the one before. Escape (`stop`) leaves the element without sending.
  - A formatting bar floats above the selection inside an editable formatted-text element, with Bold, Italic, Code and Link. Link asks for an address and keeps it only if `safeLink` accepts it.
  - While an element is being typed into, the page is not redrawn under the cursor; the redraw waits until editing stops.

- [ ] **Step 1: Write the failing tests**

```js
test('what a browser produced while typing is tidied to the allowed subset', () => {
    assert.equal(tidyTyped('<b>a</b> <i>b</i> <strike>c</strike>'), '<strong>a</strong> <em>b</em> <s>c</s>');
    assert.equal(tidyTyped('<span style="color:red">a</span><font face="x">b</font>'), 'ab');
    assert.equal(tidyTyped('one<div>two</div><div>three</div>'), 'one<br>two<br>three');
    assert.equal(tidyTyped('<a href="https://e.test" onclick="x" class="y">a</a>'), '<a href="https://e.test">a</a>');
    assert.equal(tidyTyped('a&nbsp;b<br>'), 'a b');
});
test('keys while typing decide what happens', () => {
    assert.equal(keyAction({ key: 'Enter', shiftKey: false }), 'commit-split');
    assert.equal(keyAction({ key: 'Enter', shiftKey: true }), 'line-break');
    assert.equal(keyAction({ key: 'Backspace', empty: true, atStart: true }), 'remove');
    assert.equal(keyAction({ key: 'Backspace', empty: false, atStart: true }), null);
    assert.equal(keyAction({ key: 'Escape' }), 'stop');
    assert.equal(keyAction({ key: 'z', metaKey: true }), null);
});
test('a link is kept only when it is safe', () => {
    assert.equal(safeLink('example.com/a'), 'https://example.com/a');
    assert.equal(safeLink('/about'), '/about');
    assert.equal(safeLink('javascript:alert(1)'), null);
    assert.equal(safeLink(' //evil.test '), null);
});
test('a value can be set deep inside settings without touching the rest', () => {
    const settings = { rows: [['a', 'b'], ['c', 'd']], caption: 'x' };
    assert.deepEqual(withValue(settings, 'rows.1.0', 'C'), { rows: [['a', 'b'], ['C', 'd']], caption: 'x' });
    assert.deepEqual(settings.rows[1], ['c', 'd']);
});
test('a block of the same type can be added right after another and selected', async () => {
    await builder.insertAfter('b', 'acme/box');

    assert.deepEqual(client.calls.find((call) => call[0] === 'operate')[1], {
        operation: 'insert', type: 'acme/box', position: { parent: null, index: 2 }, settings: {},
    });
    assert.equal(builder.state().selected, 'new');
});
```

- [ ] **Step 2: Run** `cd block-builder && npm test`. Expected: FAIL.
- [ ] **Step 3: Implement.**
- [ ] **Step 4: Run** `npm test`; then in the demo (Task 4 loads the text blocks, so check after it): type into a paragraph, bold a word, add a link, press Enter to get a new paragraph, Backspace it away, and edit a table cell. Expected: PASS, and the page changes as typed.
- [ ] **Step 5: Commit** `feat(block-builder): type into text blocks on the page`.

---

### Task 4: Demo, inspector controls and documentation

**Files:**
- Modify: `block-builder/demo/server.php`, `block-builder/demo/composer.json`, `block-builder/src/builder-element.js` (controls for `x-control: table` and `x-control: code`), repository `README.md`, `wordpress-develop/docs/superpowers/specs/2026-11-18-block-library-design.md` (a part 2 note)

**Interfaces:**
- The demo loads `TextTypesPackage` and `TextPackage` and seeds its page with a heading, two paragraphs, a list with a nested list, a quote, a code block, a table and a separator.
- The inspector draws `x-control: table` as row and column counts with Add row, Add column, Remove row and Remove column buttons acting on `rows`, keeping existing cell text; and `x-control: code` as a monospace text area.

- [ ] **Step 1: Write the failing tests:** `forms.test.js` gains `tableShape(rows)` (rows and columns), `withRow(rows, at)`, `withoutRow(rows, at)`, `withColumn(rows, at)`, `withoutColumn(rows, at)`, all leaving existing text in place and never returning fewer than one row or one column.
- [ ] **Step 2: Run** `npm test`. Expected: FAIL.
- [ ] **Step 3: Implement**, then update the demo and the README ("Text" under the block areas; typing on the page under the builder).
- [ ] **Step 4: Run** every PHP suite and `npm test`; start the demo and do Task 3's hand check plus adding a table row. Expected: PASS.
- [ ] **Step 5: Commit** `feat(block-text): show the text blocks in the demo and edit tables in the inspector`.

---

## Out of scope for this plan

The other areas (parts 3 to 6), media picking, the table of contents (part 6, which reads the `id` headings now compute), duplicate heading anchors across a page (recorded as a known limit), pasting blocks rather than text, and the remaining editor features (part 7).
