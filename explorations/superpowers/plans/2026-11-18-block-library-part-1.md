# Block library part 1: foundations Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Lay the contracts and seams every block area and every editor builds on: package titles and per-plugin sections, output ownership and output-declared assets, the asset registry, formatted text, editing marks, the editor seam, document steps, the block package suite's two new modes, and `block-structure` split into definitions and outputs.

**Architecture:** New contracts go in `block-domain` (zero dependencies). Defaults go in `block-app`. The builder API (`block-presentation`) learns the editing flag and package sections. `block-builder` becomes a mixed package with a small PHP side implementing the editor contract. `block-structure` becomes `block-structure-domain` (types) and `block-structure` (outputs). No block names or settings change.

**Tech Stack:** PHP 8.4, Pest, Composer path repositories; plain ES modules and `node --test` for the builder.

**Spec:** `docs/superpowers/specs/2026-11-18-block-library-design.md` (in `wordpress-develop`). The model it builds on: `docs/superpowers/specs/2026-11-17-block-model-design.md`.

**Repository:** `wordpress-develop/packages/speedyspec-api-block` (its own git repo; branch from `origin/main`). Every path below is relative to its `packages/` directory.

## Global Constraints

- `declare(strict_types=1)`; 4-space indentation; every opening brace of a class, method or control structure on its own line, multi-line signatures and empty bodies included (closures, arrow functions, anonymous classes and `match` exempt).
- Docblocks per `wordpress-develop/.claude/rules/coding-style.md`: a summary that says why and does not start with "Returns"; no `@return void`; `@since 0.1.0`; no type tag restating a non-array type the signature gives; lines at most 120 characters.
- `block-domain` has zero dependencies and no `final`. Nothing references WordPress outside `block-infra-wp`.
- `block-app`, `block-presentation`, `block-structure-domain`, `block-structure` and `block-builder`'s PHP side require only `speedyspec/block-domain` at runtime, except that `block-structure` also requires `speedyspec/block-structure-domain`.
- Each contract is bound with its own `add()` and lazy collaborators, so any one can be replaced alone.
- Block names and stored settings of existing blocks do not change; existing documents open unchanged.
- Run a package's tests with `cd packages/<pkg> && vendor/bin/pest --colors=never`; the builder's with `cd packages/block-builder && npm test`. Run `composer update` in a package whose dependencies change.
- One commit per task, with explicit `git add` paths, ending with `Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>`. Never commit `vendor/`, `composer.lock` or `.phpunit.cache/`.

## Review Focus

1. **Formatted text that is not clean** (script elements, event attributes, `javascript:` and `data:` links, unclosed and misnested tags, entities that decode to markup, very long or deeply nested input, invalid UTF-8). Expected: only the allowed subset survives, text is kept, nothing throws. Task 4.
2. **A bound value reaching a formatted-text setting.** Expected: it is drawn as escaped text, never as markup, whatever the data source returns. Task 4.
3. **Editing marks leaking to visitors** (a front render with the editing flag set, an admin render with no editor bound, a block whose output ignores the attribute bag). Expected: marks appear only in admin renders while an editor is bound. Task 5.
4. **Asset dependencies that cannot be resolved** (a cycle, a missing dependency, a name registered twice by two packages, a package file path with `..`). Expected: a clear exception at registration or resolution, never a partial or looping list, and no path outside the package. Task 3.
5. **A replacement output loaded before the default.** Expected: the last registration wins and `outputOwner()` names it; the default's assets are not collected for blocks the replacement draws. Task 2.

---

### Task 1: Package titles and sections

**Files:**
- Modify: `block-domain/src/Contracts/BlockPackageInterface.php`, `block-domain/src/Contracts/BlockPackageLoaderInterface.php`, `block-domain/src/ValueObjects/CatalogueEntry.php`
- Modify: `block-app/src/Packages/BlockPackageLoader.php`, `block-app/src/Building/BlockCatalogue.php`, every class implementing `BlockPackageInterface` (`block-app/src/Flow/FlowPackage.php`, `block-app/src/Presentation/PresentationPackage.php`, `block-app/src/Documents/DocumentsPackage.php`, `block-structure/src/StructurePackage.php`, `block-example/src/ExampleBlockPackage.php`), the `package()` helper in `block-app/tests/Pest.php` and any anonymous packages in tests
- Modify: `block-presentation/src/BuilderApi.php`
- Modify: `block-builder/src/api.js` (`catalogue()` returns `{packages, blocks}`), `block-builder/src/store.js`, `block-builder/src/view.js` (`groupCatalogue`), `block-builder/src/builder-element.js`
- Test: `block-app/tests/Packages/BlockPackageLoaderTest.php`, `block-app/tests/Building/BlockCatalogueTest.php`, `block-presentation/tests/BuilderApiTest.php`, `block-builder/tests/view.test.js`, `block-builder/tests/api.test.js`

**Interfaces:**
- Produces: `BlockPackageInterface::title(): string`, `BlockPackageInterface::description(): string`.
- Produces: `BlockPackageLoaderInterface::details(string $packageName): ?array` returning `['title' => string, 'description' => string]`, null for a package not loaded.
- Produces: `CatalogueEntry::__construct(BlockType $type, ?string $package, string $packageTitle = '')`; `toArray()` gains `packageTitle`.
- Produces: `GET /catalogue` answers `{"packages": [{"name", "title", "description"}...], "blocks": [...]}`, packages in load order, only those that defined at least one listed block.
- Changes: `groupCatalogue(entries, packages, query)` in `block-builder/src/view.js` returns sections `{package, title, description, groups: [{category, entries}]}`: one section per defining package in the order given, a block with no package in a last "Added by the site" section, categories as subgroups only when a section has more than one. Search matches a block's title, name, keywords, description and its package's title; empty sections are left out.

- [ ] **Step 1: Write the failing tests**

```php
test('a package must have a title, and the loader keeps its details', function () {
    // package('acme/one', ..., title: 'Acme blocks', description: 'Boxes.') after the helper gains two arguments
    expect($loader->details('acme/one'))->toBe(['title' => 'Acme blocks', 'description' => 'Boxes.'])
        ->and($loader->details('acme/nope'))->toBeNull()
        ->and(fn () => $loader->load(package('acme/two', $register, '  ')))->toThrow(BlockPackageException::class);
});
test('a catalogue entry names the package that defined it, by title', function () {
    expect($catalogue->find(BlockName::fromString('acme/box'))->toArray()['packageTitle'])->toBe('Acme blocks')
        ->and($catalogue->find(BlockName::fromString('host/added'))->toArray()['packageTitle'])->toBe('');
});
test('the catalogue route lists each defining package once, in load order', function () {
    expect($body['packages'])->toBe([['name' => 'core/flow', 'title' => 'Flow', 'description' => ...], ...])
        ->and(array_column($body['blocks'], 'packageTitle'))->each->toBeString();
});
```

```js
test('the add-block panel has a section per plugin, searchable across all of them', () => {
    // packages: core/flow 'Flow', shop/blocks 'Shop'; entries from both and one with package null
    assert.deepEqual(groupCatalogue(entries, packages, '').map((s) => s.title), ['Flow', 'Shop', 'Added by the site']);
    assert.deepEqual(groupCatalogue(entries, packages, 'shop').map((s) => s.title), ['Shop']); // matched by package title
    assert.deepEqual(groupCatalogue(entries, packages, 'nothing'), []);
});
```

A blank title (after trimming) is refused with `BlockPackageException` when loading.

- [ ] **Step 2: Run** `block-app`, `block-presentation` and `block-builder` tests. Expected: FAIL (missing methods).
- [ ] **Step 3: Implement.** Titles for the existing packages: Flow ("Show by rule and repeat"), Presentation ("Style settings"), Documents ("Included documents"), Structure ("Layout"), Example ("Example blocks"), each with a one-sentence description.
- [ ] **Step 4: Run** every PHP suite and `npm test`; open the demo and check the add-block panel shows a section per package. Expected: PASS.
- [ ] **Step 5: Commit** `feat(block): give block packages a title and group the add-block panel by package`.

---

### Task 2: Output ownership and output-declared assets

**Files:**
- Create: `block-domain/src/Contracts/DeclaresAssetsInterface.php`
- Modify: `block-domain/src/Contracts/BlockPackageLoaderInterface.php`, `block-app/src/Packages/BlockPackageRegistrar.php`, `block-app/src/Packages/BlockPackageLoader.php`, `block-app/src/Output/DocumentRenderer.php`
- Test: `block-app/tests/Packages/BlockPackageLoaderTest.php`, `block-app/tests/Output/DocumentRendererTest.php`

**Interfaces:**
- Produces: `interface DeclaresAssetsInterface { public function assets(): array; }` — list of asset names, implemented optionally by a `BlockOutputInterface`.
- Produces: `BlockPackageLoaderInterface::outputOwner(BlockName $type, Mode $mode, string $target = 'html'): ?string` — the package that last registered that output through a loader; null when none did.

- [ ] **Step 1: Write the failing tests**

```php
test('the last package to register an output draws the block, and the loader says which', function () {
    // load 'acme/types' (type acme/tab + output), then 'other/take' registering an output for acme/tab
    expect($loader->outputOwner(BlockName::fromString('acme/tab'), Mode::Front))->toBe('other/take')
        ->and($loader->outputOwner(BlockName::fromString('acme/tab'), Mode::Admin))->toBeNull()
        ->and($renderer->render(doc(block('t', 'acme/tab')), new RenderRequest())->output)->toBe('other');
});
test('an output names the assets it needs, and a replaced output no longer brings its own', function () {
    // default output implements DeclaresAssetsInterface with ['acme/tabs-script']; replacement declares ['other/tabs']
    expect($before->assets)->toBe(['acme/tabs-script'])->and($after->assets)->toBe(['other/tabs']);
});
test("a block type's own assets still come first, then the output's, each once", function () {
    expect($result->assets)->toBe(['type-asset', 'output-asset']);
});
```

- [ ] **Step 2: Run** `block-app` tests. Expected: FAIL.
- [ ] **Step 3: Implement.** The registrar records `[type, mode, target] => package` for outputs it commits; the loader keeps the latest. `DocumentRenderer::collect()` receives the output that drew the block and appends `assets()` when it implements `DeclaresAssetsInterface`; a non-string name is a `StepFailed` issue as for collect steps.
- [ ] **Step 4: Run** `block-app`. Expected: PASS.
- [ ] **Step 5: Commit** `feat(block): record who draws each block and let outputs declare their assets`.

---

### Task 3: Asset registry and locator

**Files:**
- Create in `block-domain/src/`: `Contracts/AssetRegistryInterface.php`, `Contracts/AssetLocatorInterface.php`, `ValueObjects/Asset.php`, `ValueObjects/AssetKind.php`, `Exceptions/InvalidAssetException.php`
- Create: `block-app/src/Assets/AssetRegistry.php`
- Modify: `block-domain/src/Contracts/BlockPackageRegistrarInterface.php`, `block-app/src/Packages/BlockPackageRegistrar.php`, `block-app/src/AppServiceProvider.php`
- Test: `block-domain/tests/ValueObjects/AssetTest.php`, `block-app/tests/Assets/AssetRegistryTest.php`

**Interfaces:**
- Produces: `enum AssetKind: string { case Script = 'script'; case Style = 'style'; }`.
- Produces: `Asset` (readonly) with `static inline(string $name, AssetKind $kind, string $content, array $dependencies = [])`, `static url(string $name, AssetKind $kind, string $url, array $dependencies = [])`, `static file(string $name, AssetKind $kind, string $package, string $path, array $dependencies = [])`; properties `name`, `kind`, `source` (`'inline'|'url'|'file'`), `content`, `url`, `package`, `path`, `dependencies`. Names are `vendor/name`; a file path is relative, with no `..` segment and no leading `/`; a URL is `http`, `https` or starts with `/`.
- Produces: `AssetRegistryInterface`: `add(Asset $asset): void` (replaces one of the same name), `get(string $name): ?Asset`, `resolve(array $names): array` — the named assets and their dependencies, dependencies first, each once; throws `InvalidAssetException` naming the missing or circular asset.
- Produces: `AssetLocatorInterface::url(string $package, string $path): string`, implemented by the host. No default is bound.
- Produces: `BlockPackageRegistrarInterface::asset(Asset $asset): void` — the asset's vendor must match the package's (`BlockPackageException` otherwise), committed with the package.

- [ ] **Step 1: Write the failing tests**

```php
test('an asset refuses a name, path or URL that could reach somewhere else', function (callable $make) {
    expect($make)->toThrow(InvalidAssetException::class);
})->with([
    fn () => Asset::file('Bad Name', AssetKind::Script, 'acme/blocks', 'tabs.js'),
    fn () => Asset::file('acme/tabs', AssetKind::Script, 'acme/blocks', '../secrets.php'),
    fn () => Asset::file('acme/tabs', AssetKind::Script, 'acme/blocks', '/etc/passwd'),
    fn () => Asset::url('acme/tabs', AssetKind::Script, 'javascript:alert(1)'),
]);
test('resolving gives dependencies first, each once, in a stable order', function () {
    // a depends on c, b depends on c and a
    expect(array_map(fn (Asset $x) => $x->name, $registry->resolve(['acme/b', 'acme/a'])))
        ->toBe(['acme/c', 'acme/a', 'acme/b']);
});
test('a missing or circular dependency is named', function () {
    expect(fn () => $registry->resolve(['acme/loop']))->toThrow(InvalidAssetException::class, 'acme/loop')
        ->and(fn () => $registry->resolve(['acme/needs-gone']))->toThrow(InvalidAssetException::class, 'acme/gone');
});
test('a package registers its assets with the rest of its load, under its own vendor', function () {
    expect($registry->get('acme/tabs'))->not->toBeNull()
        ->and(fn () => $loader->load($otherVendorAsset))->toThrow(BlockPackageException::class)
        ->and($registry->get('other/leftover'))->toBeNull(); // a failed load leaves no asset
});
```

- [ ] **Step 2: Run** `block-domain` and `block-app`. Expected: FAIL.
- [ ] **Step 3: Implement.** Bind `AssetRegistryInterface` to `AssetRegistry` in `AppServiceProvider`; the loader passes it to the registrar.
- [ ] **Step 4: Run** both. Expected: PASS.
- [ ] **Step 5: Commit** `feat(block): add the asset registry and locator contracts`.

---

### Task 4: Formatted text

**Files:**
- Create: `block-domain/src/Contracts/RichTextSanitizerInterface.php`, `block-app/src/Output/RichTextSanitizer.php`
- Modify: `block-domain/src/Contracts/OutputEscaperInterface.php` (gains `richText(string $value): string`), `block-app/src/Output/HtmlEscaper.php`, `block-app/src/AppServiceProvider.php`, `block-app/src/Building/DocumentEditor.php`, `block-app/src/Resolving/ResolveStage.php`, `block-contract-tests/src/outputEscaperContract.php`
- Test: `block-app/tests/Output/RichTextSanitizerTest.php`, additions to `block-app/tests/Building/DocumentEditorTest.php` and `block-app/tests/Resolving/ResolveStageTest.php`

**Interfaces:**
- Produces: `RichTextSanitizerInterface::clean(string $html): string`.
- Produces: `OutputEscaperInterface::richText(string $value): string` — for `html`, the cleaned value; an escaper for another target writes the text without markup. This is the "helper on `RenderInput`" the spec names: outputs reach it as `$input->escaper->richText()`.
- Rule: a setting is formatted text when its schema is `type: string` with `x-control: rich-text`.

- [ ] **Step 1: Write the failing tests**

```php
test('only the allowed subset survives cleaning, and text is kept', function (string $in, string $out) {
    expect($sanitizer->clean($in))->toBe($out);
})->with([
    ['<strong>a</strong> <em>b</em>', '<strong>a</strong> <em>b</em>'],
    ['<script>x()</script>ok', 'x()ok'],
    ['<b onclick="x">a</b>', 'a'],
    ['<a href="javascript:x">a</a>', 'a'],
    ['<a href="https://e.test/?q=1" target="_blank">a</a>', '<a href="https://e.test/?q=1">a</a>'],
    ['<a href="/about">a</a>', '<a href="/about">a</a>'],
    ['<em><strong>a</em></strong>', '<em><strong>a</strong></em>'],
    ['&lt;script&gt;', '&lt;script&gt;'],
    ['a<br>b<br/>', 'a<br>b<br>'],
    ['<!-- c -->a', 'a'],
    ["\xC3\x28", "\u{FFFD}("],
]);
test('cleaning never throws and stays bounded on hostile input', function () {
    expect($sanitizer->clean(str_repeat('<em>', 100000) . 'x'))->toContain('x');
});
test('the editor stores formatted text already cleaned', function () {
    expect($result->document->tree->find('p')->settings['text'])->toBe('<strong>a</strong>');
});
test('a bound value into formatted text is drawn as text', function () {
    // source returns '<img src=x onerror=1>'
    expect($output)->toBe('<p>&lt;img src=x onerror=1&gt;</p>');
});
```

The escaper contract suite gains: `richText()` never returns a `<script` or an `on*=` attribute for any of the cases above.

- [ ] **Step 2: Run** `block-app`. Expected: FAIL.
- [ ] **Step 3: Implement.** The sanitizer uses `DOMDocument` (in `ext-dom`, declared in `block-app`'s `require`) or an equivalent tokenizer; pick one and keep it bounded (cap nesting depth at 32, deeper elements become text). `ResolveStage` escapes a bound value to HTML text before it lands in a formatted-text setting. `DocumentEditor` cleans formatted-text settings in `insert`, `insertBlocks`, `restore` and `updateSettings` before validating.
- [ ] **Step 4: Run** every PHP suite. Expected: PASS.
- [ ] **Step 5: Commit** `feat(block): add formatted text and clean it on every way in`.

---

### Task 5: Editing marks and the editor seam

**Files:**
- Create in `block-domain/src/`: `Contracts/EditorInterface.php`, `ValueObjects/EditorRequest.php`, `ValueObjects/EditorPage.php`
- Modify: `block-domain/src/ValueObjects/RenderRequest.php`, `block-domain/src/ValueObjects/RenderInput.php`, `block-presentation/src/BuilderAnnotations.php`, `block-presentation/src/BuilderApi.php`
- Create: `block-builder/composer.json`, `block-builder/phpunit.xml`, `block-builder/php/ReferenceEditor.php`, `block-builder/php/ReferenceEditorServiceProvider.php`, `block-builder/tests-php/ReferenceEditorTest.php`
- Modify: `block-builder/demo/server.php`, `block-builder/demo/composer.json`, `block-builder/demo/index.html` (or remove it in favour of the editor's page)
- Test: `block-presentation/tests/BuilderApiTest.php`, `block-builder/tests-php/ReferenceEditorTest.php`

**Interfaces:**
- Produces: `RenderRequest::__construct(Mode $mode = Mode::Front, string $target = 'html', bool $editing = false)`; `editing` is ignored (treated as false) unless `mode` is `Admin`.
- Produces: `RenderInput::editable(string $setting): AttributeBag` — `$this->attributes` with `data-block-setting` set to the setting name when `request->editing` is true; `$this->attributes` unchanged otherwise.
- Produces: `EditorInterface`: `name(): string`, `title(): string`, `open(EditorRequest $request): EditorPage`.
- Produces: `EditorRequest` (readonly): `documentId`, `apiUrl`, `array $values` (host values such as a token).
- Produces: `EditorPage` (readonly): `array $assets` (asset names, resolved by the host through the registry and locator), `string $markup`.
- Changes: `BuilderAnnotations` adds `data-block-id` only when `request->editing` is true. `BuilderApi` sets `editing` on admin renders and previews when `EditorInterface` is bound.
- Produces: `block-builder` PHP namespace `SpeedySpec\Block\Builder\`, `ReferenceEditor` registering its scripts as file assets of package `speedyspec/block-builder` (`src/builder-element.js` and the modules it imports) and returning markup `<block-builder api="…" document="…"></block-builder>` plus an inline module that calls `defineBlockBuilder()`, every value escaped.

- [ ] **Step 1: Write the failing tests**

```php
test('with no editor bound, admin renders carry no editing marks', function () {
    expect($admin['output'])->toBe('<div><p>Hi</p></div>');
});
test('with an editor bound, admin renders and previews carry block ids, visitors never do', function () {
    expect($admin['output'])->toBe('<div data-block-id="b"><p data-block-id="t">Hi</p></div>')
        ->and($front['output'])->toBe('<div><p>Hi</p></div>');
});
test('an output marks the setting a person can type into, only while editing', function () {
    expect($editing)->toContain('data-block-setting="text"')->and($notEditing)->not->toContain('data-block-setting');
});
test('the reference editor names its scripts as assets and escapes what it places', function () {
    $page = $editor->open(new EditorRequest('a"b', '/api', []));
    expect($page->assets)->toContain('speedyspec/block-builder')
        ->and($page->markup)->toContain('document="a&quot;b"')->not->toContain('a"b"');
});
```

The existing test "an editor's drawing names each block on its element" is updated to bind an editor first.

- [ ] **Step 2: Run** `block-presentation` and the new `block-builder` PHP tests. Expected: FAIL.
- [ ] **Step 3: Implement.** `block-builder/composer.json` requires `php` and `speedyspec/block-domain`, with `block-app`, `block-infra-memory` and Pest as dev requirements and the path repository `../*`. The demo binds the reference editor, an `AssetLocatorInterface` that maps `speedyspec/block-builder` files to `/src/…`, and draws the editor page instead of the static `index.html`.
- [ ] **Step 4: Run** `block-presentation`, `block-builder` (PHP and `npm test`). Start the demo and check by hand that the editor opens, a block can be clicked, and the front render has no marks. Expected: PASS.
- [ ] **Step 5: Commit** `feat(block): add the editor seam and tie editing marks to it`.

---

### Task 6: Document steps

**Files:**
- Create: `block-domain/src/Contracts/DocumentStepInterface.php`
- Modify: `block-app/src/Resolving/DocumentResolver.php`, `block-app/src/AppServiceProvider.php`, `block-domain/src/Contracts/BlockPackageRegistrarInterface.php` docblock (steps may now be document steps)
- Test: `block-app/tests/Resolving/DocumentStepTest.php`

**Interfaces:**
- Produces: `DocumentStepInterface::context(Document $document, Context $context, Mode $mode): Context` — runs once per resolve, before any block, and returns the context the root blocks get.
- Changes: `DocumentResolver::__construct(..., int $budget = 50000, ?StepRegistryInterface $steps = null)`; with steps, every registered `DocumentStepInterface` runs in priority order. A step that throws is skipped and recorded as a `StepFailed` issue with an empty block id.

- [ ] **Step 1: Write the failing tests**

```php
test('a document step sees the whole document once and gives the root blocks context', function () {
    // step counts headings and sets 'acme/count'
    expect($resolved->find('child')->context->get('acme/count'))->toBe(2)->and($calls)->toBe(1);
});
test('a failing document step is recorded and the document still resolves', function () {
    expect(idsAndKinds($resolved->issues))->toBe([['', IssueKind::StepFailed]])
        ->and($resolved->find('a')->isResolved())->toBeTrue();
});
```

Context keys reach a block only if its type `uses` them; the test type declares `uses: ['acme/count']`.

- [ ] **Step 2: Run** `block-app`. Expected: FAIL.
- [ ] **Step 3: Implement.**
- [ ] **Step 4: Run** `block-app`. Expected: PASS.
- [ ] **Step 5: Commit** `feat(block): add document steps`.

---

### Task 7: The block package suite's two modes

**Files:**
- Modify: `block-contract-tests/src/blockPackageContract.php`
- Test: `block-app/tests/Packages/BlockPackageContractFailureTest.php`

**Interfaces:**
- Produces: `blockPackageContract(Closure $makePackage, string $composerJsonPath, array $with = [], string $kind = 'complete')` with `$kind` one of `complete` (today's checks), `definitions` (no output checks; still title, example, schema, examples resolve without issues in both modes, private settings absent from the default view in front mode) and `outputs` (`$with` must load the definitions package; every type of those packages has front html output, examples draw without issues in both modes, text is escaped, private and admin values stay out of front output; the package under test itself defines no types).
- Changes: the runtime rule allows `speedyspec/block-domain` and any `speedyspec/*-domain` package.
- Produces: a check in all modes that the package's `title()` is not blank.

- [ ] **Step 1: Write the failing tests** in `BlockPackageContractFailureTest.php`: a definitions package passes in `definitions` mode and fails in `complete` mode; an outputs package missing one output fails in `outputs` mode naming that type; an outputs package requiring `speedyspec/block-structure-domain` passes the runtime rule; one requiring `speedyspec/block-app` does not.
- [ ] **Step 2: Run** `block-app`. Expected: FAIL.
- [ ] **Step 3: Implement.**
- [ ] **Step 4: Run** every package that runs the suite (`block-app`, `block-example`, `block-structure`). Expected: PASS.
- [ ] **Step 5: Commit** `feat(block): let the package suite check definitions and outputs packages apart`.

---

### Task 8: Split `block-structure`

**Files:**
- Create: `block-structure-domain/` (`composer.json`, `phpunit.xml`, `src/StructureTypesPackage.php`, `src/Layout.php` if the types need it, `tests/Pest.php`, `tests/PackageContractTest.php`)
- Modify: `block-structure/` (`composer.json` requires `speedyspec/block-structure-domain`; `src/StructurePackage.php` registers only outputs and assets; `tests/`), `block-builder/demo/server.php`, `block-builder/demo/composer.json`, repository `README.md`
- Test: both packages' suites

**Interfaces:**
- Produces: `SpeedySpec\Block\Structure\Domain\StructureTypesPackage` (name `core/structure-types`, title "Layout") registering the five block types with exactly today's names and settings, and no outputs.
- Changes: `SpeedySpec\Block\Structure\StructurePackage` (name `core/structure`, title "Layout blocks, drawn") registers the outputs; `from($container)` unchanged.
- The add-block panel's section for these blocks is the definitions package's.

- [ ] **Step 1: Write the failing tests:** `block-structure-domain` runs the suite in `definitions` mode; `block-structure` runs it in `outputs` mode with the types package in `$with`; a document stored before the split (the existing `OutputTest` fixtures) draws the same markup and stylesheet.
- [ ] **Step 2: Run** both. Expected: FAIL.
- [ ] **Step 3: Implement.**
- [ ] **Step 4: Run** both and the demo by hand. Expected: PASS; the demo's page opens unchanged.
- [ ] **Step 5: Commit** `refactor(block-structure): split the layout blocks into definitions and outputs`.

---

## Out of scope for this plan

Every block area (parts 2 to 6), the media library contract (part 3), typing on the page in the builder (part 2), the remaining editor features (part 7), and any WordPress admin wiring.
