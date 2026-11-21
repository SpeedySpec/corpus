# Block System Part 1b (output and the block package contract) Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Draw a resolved document as HTML in front or admin mode, with each block's output supplied by the package that adds the block, and define the entry point and shared test suite that make a block package portable.

**Architecture:** Builds on part 1a in `packages/speedyspec-api-block/`. `block-domain` gains the output contracts (escaper, attribute bag, block output, render and collect steps, document renderer) and the block package contracts. `block-app` gains their framework-free defaults: an HTML escaper, an output registry that falls back from admin to front, a document renderer that runs stages 6 (render) and 7 (collect) over a resolved document, and a package loader. `block-contract-tests` gains the suite every block package runs. A new `block-example` package is a small real block package that proves the suite and shows how one is written.

**Tech Stack:** PHP 8.4, Composer path repositories, Pest 3/4.

**Spec:** `docs/superpowers/specs/2026-11-17-block-model-design.md` (in the `wordpress-develop` checkout). Sections "Mode", "Behaviour and view", "Output and attributes", "The three roles" and stages 6 and 7 of "Resolving and rendering" are what this plan implements.

## Global Constraints

- Work only inside `/Users/jacobsantos/repos/respite-studios/wordpress-develop/packages/speedyspec-api-block/`, a clone of `SpeedySpec/speedyspec-api-block`. Never run git against the surrounding `wordpress-develop` repository and never change files outside this directory. Never push.
- Branch `feat/block-phase-1b`, cut from `feat/block-phase-1a` (part 1a's pull request may still be open; this branch stacks on it). One commit per task. Never commit `vendor/`, `composer.lock` or `.phpunit.cache/`.
- PHP `>=8.4`; every PHP file starts with `declare(strict_types=1);`; 4-space indentation.
- Braces: the opening brace of every class, interface, enum, function, method and control structure (`if`, `else`, `foreach`, `for`, `while`, `switch`, `try`, `catch`) goes on its own line, including multi-line signatures and empty bodies (never `) {`, `): Type {` or `{}` on a signature line). Closures, arrow functions and anonymous classes keep their brace on the same line.
- Docblocks follow `/Users/jacobsantos/repos/respite-studios/wordpress-develop/.claude/rules/coding-style.md`: a summary line that does not start with "Returns", the "why" not the "how", no `@return void`, `@since 0.1.0`, no `@param`/`@return` that only restates a non-array type, lines at most 120 characters.
- `block-domain` requires only PHP and has no `final` classes. No package references a WordPress function, class or package.
- `block-app`, `block-infra-memory` and `block-example` require only `speedyspec/block-domain` at runtime. `block-contract-tests` requires `speedyspec/block-domain`, `speedyspec/block-app`, `speedyspec/block-infra-memory` and Pest (it is a development tool; this is the one package allowed to require implementations).
- No `src` file names a class from another implementation package. Services reach each other only through `BlockServiceContainer`.
- Namespaces: `SpeedySpec\Block\Domain\`, `SpeedySpec\Block\App\`, `SpeedySpec\Block\Infra\Memory\`, `SpeedySpec\Block\ContractTests\`, `SpeedySpec\Block\Example\`. Composer packages use `"version": "0.1.0"`, license `GPL-2.0-or-later`, path repository `../*`, siblings required as `"^0.1"`, and the same `archive`, `config` and `phpunit.xml` as `packages/block-app`.
- Tests are Pest, files end `Test.php`, run with `vendor/bin/pest --colors=never` inside the package. Output must be free of warnings and deprecations. After changing a sibling's `composer.json` or adding a file to an `autoload.files` list, run `composer update <that package>` in each dependent package (`composer dump-autoload` alone is not enough).
- Rendering never throws for bad content: it records an `Issue` and continues. A value reaches output only through the escaper or the attribute bag.
- The one output target in this part is `'html'`.

## Review Focus

Each line has a pinning test in the task named.

1. **Markup in a setting or bound value** (`<script>`, `"` in an attribute, `javascript:` in a URL). Expected: escaped for its context, or dropped for an unsafe URL scheme; never emitted as markup. Tasks 1 and 5.
2. **A decorator or step trying to inject an attribute that executes script** (`onclick`, `style` with `url(javascript:…)` or `expression(`, an attribute name with a space or `>`). Expected: rejected when added to the attribute bag. Task 1.
3. **Admin-only data in front output** (a private setting, a behaviour's admin context value, a block's admin output). Expected: absent from anything rendered in front mode. Tasks 3 and 5.
4. **A block type with no admin output, and one with no output at all.** Expected: admin mode uses the front output; a type with no output records `MissingOutput` and its children's output still appears. Tasks 2 and 3.
5. **A block output that throws.** Expected: that block yields nothing, `OutputFailed` is recorded, siblings and the rest of the page render. Task 3.

---

## File Structure

```
packages/
├── block-domain/src/
│   ├── Contracts/     OutputEscaperInterface, OutputEscaperRegistryInterface, BlockOutputInterface,
│   │                  BlockOutputRegistryInterface, RenderStepInterface, CollectStepInterface,
│   │                  DocumentRendererInterface, BlockPackageInterface, BlockPackageRegistrarInterface,
│   │                  BlockPackageLoaderInterface
│   ├── ValueObjects/  AttributeBag, RenderInput, RenderRequest, RenderResult (IssueKind gains three cases)
│   └── Exceptions/    InvalidAttributeException, UnsupportedTargetException
├── block-app/src/
│   ├── Output/        HtmlEscaper, OutputEscaperRegistry, BlockOutputRegistry, DocumentRenderer
│   ├── Packages/      BlockPackageRegistrar, BlockPackageLoader
│   └── AppServiceProvider.php (new bindings)
├── block-contract-tests/src/   outputEscaperContract.php, blockPackageContract.php
└── block-example/     composer.json, phpunit.xml, README.md,
    ├── src/           ExampleBlockPackage, Notice/{NoticeOutput,NoticeAdminOutput},
    │                  Listing/{ListOutput,ListItemOutput,ListBehaviour}
    └── tests/         PackageContractTest.php, OutputTest.php
```

---

### Task 1: Escaping and the attribute bag

**Files:**
- Create in `block-domain/src/`: `ValueObjects/AttributeBag.php`, `Contracts/{OutputEscaperInterface,OutputEscaperRegistryInterface}.php`, `Exceptions/{InvalidAttributeException,UnsupportedTargetException}.php`
- Create in `block-app/src/Output/`: `HtmlEscaper.php`, `OutputEscaperRegistry.php`
- Create: `block-contract-tests/src/outputEscaperContract.php` (add to `autoload.files`)
- Test: `block-domain/tests/ValueObjects/AttributeBagTest.php`, `block-app/tests/Output/HtmlEscaperTest.php`

**Interfaces:**
- `AttributeBag` (immutable): `static empty(): self`, `withClass(string ...$classes): self`, `withAttribute(string $name, string $value): self`, `withStyle(string $property, string $value): self`, `withId(string $id): self`, `merge(self $other): self` (classes unioned in order, other's attributes, styles and id win), `classes(): array` (unique, insertion order), `attributes(): array` (name to value), `styles(): array` (property to value), `id(): ?string`, `isEmpty(): bool`.
- `AttributeBag` rejects, with `InvalidAttributeException` (extends `\InvalidArgumentException`):
  - a class that is empty or contains whitespace, `"`, `'`, `<`, `>`, `=` or a control character;
  - an attribute name not matching `/^[a-zA-Z_][a-zA-Z0-9_.:-]*\z/`, or equal (case-insensitively) to `class`, `id` or `style`, or starting (case-insensitively) with `on`;
  - a style property not matching `/^-{0,2}[a-zA-Z][a-zA-Z0-9-]*\z/`;
  - a style value that is empty or contains `;`, `{`, `}`, `<`, `>`, a control character, or (case-insensitively) `expression(`, `javascript:` or `@import`;
  - an id that is empty or contains whitespace or a control character.
  Attribute values are stored as given; they are escaped when rendered.
- `OutputEscaperInterface`: `target(): string`, `text(string $value): string`, `attribute(string $value): string`, `url(string $value): string`, `attributes(AttributeBag $bag): string`.
- `OutputEscaperRegistryInterface`: `add(OutputEscaperInterface $escaper): void` (replaces a same-target one), `get(string $target): OutputEscaperInterface` (throws `UnsupportedTargetException`, which extends `\LogicException`, when none is registered), `has(string $target): bool`.
- `HtmlEscaper implements OutputEscaperInterface` (`target()` is `'html'`):
  - `text()` and `attribute()`: `htmlspecialchars($value, ENT_QUOTES | ENT_SUBSTITUTE | ENT_HTML5, 'UTF-8')` (existing entities ARE encoded again: stored settings are data, not markup).
  - `url()`: trims; removes ASCII control characters and whitespace before the scheme check; allows a URL with no scheme (relative, `#fragment`, `?query`, `//host`), or scheme `http`, `https`, `mailto` or `tel` (case-insensitive); anything else returns `''`. The allowed URL is returned attribute-escaped.
  - `attributes()`: returns `''` for an empty bag; otherwise a string starting with a space, in the order `id`, `class`, `style`, then the other attributes in insertion order, each `name="escaped value"`; classes joined by a space; styles joined as `property: value` with `; `.
- `outputEscaperContract(Closure $make)` asserts for any escaper: `text('<b>"&\'')` contains none of `<`, `>`, `"`; `attribute()` of a value containing `"` and a newline contains no raw `"`; `url('javascript:alert(1)')` is `''` and `url("java\tscript:alert(1)")` is `''`; `attributes(AttributeBag::empty())` is `''`; `attributes()` of a bag whose attribute value is `"><script>` contains no `<script>`.

- [ ] **Step 1: Write the failing tests**

```php
// block-domain/tests/ValueObjects/AttributeBagTest.php
test('collects classes, attributes, styles and an id immutably', function () {
    $empty = AttributeBag::empty();
    $bag = $empty->withClass('a', 'b', 'a')->withAttribute('data-x', '1')->withStyle('color', 'red')->withId('top');
    expect($empty->isEmpty())->toBeTrue()->and($bag->classes())->toBe(['a', 'b'])
        ->and($bag->attributes())->toBe(['data-x' => '1'])->and($bag->styles())->toBe(['color' => 'red'])->and($bag->id())->toBe('top');
});
test('merge unions classes and lets the other bag win elsewhere', function () {
    $a = AttributeBag::empty()->withClass('a')->withAttribute('data-x', '1')->withId('one');
    $b = AttributeBag::empty()->withClass('b', 'a')->withAttribute('data-x', '2')->withId('two');
    $m = $a->merge($b);
    expect($m->classes())->toBe(['a', 'b'])->and($m->attributes())->toBe(['data-x' => '2'])->and($m->id())->toBe('two');
});
test('rejects attribute names that could execute or break out', function (string $name) {
    expect(fn () => AttributeBag::empty()->withAttribute($name, 'x'))->toThrow(InvalidAttributeException::class);
})->with(['onclick', 'ONLOAD', 'style', 'Class', 'id', 'a b', 'a>', '', '1x', "x\n", 'x"y']);
test('rejects style properties and values that could execute or break out', function (string $property, string $value) {
    expect(fn () => AttributeBag::empty()->withStyle($property, $value))->toThrow(InvalidAttributeException::class);
})->with([
    ['color', 'red; background: url(x)'], ['color', 'expression(alert(1))'], ['background', 'url(JavaScript:alert(1))'],
    ['color', '</style>'], ['color', ''], ['col or', 'red'], ['color', "red\n"], ['color', '@import "x"'],
]);
test('rejects unsafe classes and ids', function () {
    expect(fn () => AttributeBag::empty()->withClass('a b'))->toThrow(InvalidAttributeException::class)
        ->and(fn () => AttributeBag::empty()->withClass('a"'))->toThrow(InvalidAttributeException::class)
        ->and(fn () => AttributeBag::empty()->withId('a b'))->toThrow(InvalidAttributeException::class)
        ->and(fn () => AttributeBag::empty()->withId(''))->toThrow(InvalidAttributeException::class);
});

// block-app/tests/Output/HtmlEscaperTest.php
describe('HtmlEscaper', function () {
    outputEscaperContract(fn () => new HtmlEscaper());
});
test('text and attribute values are encoded, including existing entities', function () {
    $e = new HtmlEscaper();
    expect($e->text('<b>Tom & "Jerry" &amp;</b>'))->toBe('&lt;b&gt;Tom &amp; &quot;Jerry&quot; &amp;amp;&lt;/b&gt;')
        ->and($e->attribute("it's"))->toBe('it&apos;s');
});
test('urls keep safe schemes and relative forms and drop everything else', function (string $in, string $out) {
    expect((new HtmlEscaper())->url($in))->toBe($out);
})->with([
    ['https://example.test/a?b=1&c=2', 'https://example.test/a?b=1&amp;c=2'], ['/path', '/path'], ['#top', '#top'],
    ['//cdn.example.test/x', '//cdn.example.test/x'], ['mailto:a@example.test', 'mailto:a@example.test'],
    ['tel:+15550100', 'tel:+15550100'], ['javascript:alert(1)', ''], ['JaVaScRiPt:alert(1)', ''],
    ["java\nscript:alert(1)", ''], [' javascript:alert(1)', ''], ['data:text/html,<script>', ''], ['vbscript:x', ''],
]);
test('attributes render in a fixed order with every value escaped', function () {
    $bag = AttributeBag::empty()->withAttribute('data-x', '"><script>')->withClass('a', 'b')->withStyle('color', 'red')
        ->withStyle('margin', '0')->withId('top')->withAttribute('aria-label', 'A & B');
    expect((new HtmlEscaper())->attributes($bag))
        ->toBe(' id="top" class="a b" style="color: red; margin: 0" data-x="&quot;&gt;&lt;script&gt;" aria-label="A &amp; B"');
});
test('the escaper registry replaces by target and throws for an unknown target', function () {
    $r = new OutputEscaperRegistry();
    $r->add(new HtmlEscaper());
    $mine = new HtmlEscaper();
    $r->add($mine);
    expect($r->get('html'))->toBe($mine)->and($r->has('pdf'))->toBeFalse()
        ->and(fn () => $r->get('pdf'))->toThrow(UnsupportedTargetException::class);
});
```

- [ ] **Step 2: Create the branch** (`git switch -c feat/block-phase-1b` from `feat/block-phase-1a`), add the contract file to `block-contract-tests`' `autoload.files`, run `composer update speedyspec/block-contract-tests` in `block-app`, then `vendor/bin/pest --colors=never` in `block-domain` and `block-app`. Expected: FAIL, classes not found.
- [ ] **Step 3: Implement** the value object, contracts, escaper, registry and contract function.
- [ ] **Step 4: Run `vendor/bin/pest --colors=never` in `block-domain` and `block-app`.** Expected: PASS.
- [ ] **Step 5: Commit** `feat(block): add output escaping and the attribute bag`.

---

### Task 2: Output contracts and the output registry

**Files:**
- Create in `block-domain/src/`: `ValueObjects/{RenderInput,RenderRequest,RenderResult}.php`, `Contracts/{BlockOutputInterface,BlockOutputRegistryInterface,RenderStepInterface,CollectStepInterface,DocumentRendererInterface}.php`
- Modify: `block-domain/src/ValueObjects/IssueKind.php` (add `MissingOutput = 'missing-output'`, `OutputFailed = 'output-failed'`, `InvalidAttribute = 'invalid-attribute'`)
- Create: `block-app/src/Output/BlockOutputRegistry.php`
- Test: `block-app/tests/Output/BlockOutputRegistryTest.php`, `block-domain/tests/ValueObjects/RenderValueObjectsTest.php`

**Interfaces:**
- `RenderRequest` (readonly): `__construct(Mode $mode = Mode::Front, string $target = 'html')`.
- `RenderInput` (readonly): `__construct(string $blockId, BlockType $type, BlockView $view, AttributeBag $attributes, array $childPasses, Context $context, RenderRequest $request, OutputEscaperInterface $escaper, ?AdminContext $admin = null)`. `$childPasses` is a list of strings, one already-rendered string per child pass. `children(): string` joins them. `$admin` is null in front mode.
- `BlockOutputInterface`: `render(RenderInput $input): string`.
- `BlockOutputRegistryInterface`: `set(BlockName $type, BlockOutputInterface $output, Mode $mode = Mode::Front, string $target = 'html'): void` (replaces), `get(BlockName $type, Mode $mode, string $target = 'html'): ?BlockOutputInterface` (for `Mode::Admin`, returns the admin output when one is set, otherwise the front output), `has(BlockName $type, Mode $mode, string $target = 'html'): bool` (exact mode, no fallback), `remove(BlockName $type, Mode $mode, string $target = 'html'): void`.
- `RenderStepInterface`: `attributes(AttributeBag $bag, ResolvedBlock $block, RenderRequest $request): AttributeBag` — a package that adds a decorator registers one to turn the decorator's values into classes, attributes or styles.
- `CollectStepInterface`: `assets(ResolvedBlock $block, RenderRequest $request): array` (list of asset name strings).
- `RenderResult` (readonly): `__construct(string $output, ResolvedDocument $resolved, array $issues, array $assets)`. `$issues` is every issue from resolving followed by every issue from rendering; `$assets` is a list of unique asset names in order of first appearance. `hasIssues(): bool`.
- `DocumentRendererInterface`: `render(Document $document, RenderRequest $request, ?Context $context = null): RenderResult` and `renderResolved(ResolvedDocument $resolved, RenderRequest $request): RenderResult` (throws `\InvalidArgumentException` when `$resolved->mode` differs from `$request->mode`).

- [ ] **Step 1: Write the failing tests**

```php
// helper in the test file: output(string $text): BlockOutputInterface => anonymous class returning $text
test('admin falls back to the front output and prefers an admin one', function () {
    $r = new BlockOutputRegistry();
    $name = BlockName::fromString('acme/box');
    $front = output('front');
    $r->set($name, $front);
    expect($r->get($name, Mode::Admin))->toBe($front)->and($r->has($name, Mode::Admin))->toBeFalse();
    $admin = output('admin');
    $r->set($name, $admin, Mode::Admin);
    expect($r->get($name, Mode::Admin))->toBe($admin)->and($r->get($name, Mode::Front))->toBe($front);
});
test('front never falls back to an admin output', function () {
    $r = new BlockOutputRegistry();
    $name = BlockName::fromString('acme/box');
    $r->set($name, output('admin'), Mode::Admin);
    expect($r->get($name, Mode::Front))->toBeNull();
});
test('outputs are kept per target and can be replaced and removed', function () {
    $r = new BlockOutputRegistry();
    $name = BlockName::fromString('acme/box');
    $r->set($name, output('one'));
    $two = output('two');
    $r->set($name, $two);
    expect($r->get($name, Mode::Front))->toBe($two)->and($r->get($name, Mode::Front, 'text'))->toBeNull();
    $r->remove($name, Mode::Front);
    expect($r->get($name, Mode::Front))->toBeNull();
});
// block-domain/tests/ValueObjects/RenderValueObjectsTest.php
test('a render request defaults to front html', function () {
    $r = new RenderRequest();
    expect($r->mode)->toBe(Mode::Front)->and($r->target)->toBe('html');
});
test('render input joins child passes in order', function () {
    expect(renderInput(childPasses: ['<li>a</li>', '<li>b</li>'])->children())->toBe('<li>a</li><li>b</li>')
        ->and(renderInput(childPasses: [])->children())->toBe('');
});
```

`renderInput(...)` is a `block-domain/tests/Pest.php` helper building a `RenderInput` with defaults (an `acme/box` type, an empty view, an empty bag, an empty context, a default request, and an anonymous escaper that returns its input) and named-argument overrides.

- [ ] **Step 2: Run `vendor/bin/pest --colors=never` in `block-domain` and `block-app`.** Expected: FAIL, classes not found.
- [ ] **Step 3: Implement** the value objects, contracts, enum cases and registry.
- [ ] **Step 4: Run `vendor/bin/pest --colors=never` in both packages.** Expected: PASS.
- [ ] **Step 5: Commit** `feat(block): add output contracts and the output registry`.

---

### Task 3: The document renderer (stages 6 and 7)

**Files:**
- Create: `block-app/src/Output/DocumentRenderer.php`
- Modify: `block-app/src/AppServiceProvider.php` (bind `OutputEscaperRegistryInterface` with `HtmlEscaper` added, `BlockOutputRegistryInterface`, `DocumentRendererInterface`), `block-app/src/Resolving/ViewStage.php` (enforce decorator view schemas, below)
- Test: `block-app/tests/Output/DocumentRendererTest.php`, additions to `block-app/tests/Resolving/ViewStageTest.php` and `tests/AppServiceProviderTest.php`

**Interfaces:**
- `DocumentRenderer implements DocumentRendererInterface`: `__construct(DocumentResolverInterface $resolver, BlockOutputRegistryInterface $outputs, OutputEscaperRegistryInterface $escapers, StepRegistryInterface $steps)`.
- New bindings resolve their collaborators lazily inside the factory, one `add()` per contract.

**Rules the tests pin:**
- `render()` resolves the document in the request's mode, then calls `renderResolved()`.
- `renderResolved()` gets the escaper for the request's target first; an unknown target throws `UnsupportedTargetException` (a configuration error, not bad content).
- For each resolved block, depth first:
  - A block with no view (`isResolved()` false) yields `''` and its children are not drawn. No new issue: resolving already recorded why.
  - **Stage 6.** Start from `AttributeBag::empty()`. Run each registered `RenderStepInterface` in order; a step that throws, or returns something that is not an `AttributeBag`, records `StepFailed` and leaves the bag as it was; an `InvalidAttributeException` from a step records `InvalidAttribute` instead. Render each child pass to a string (children in order, concatenated). Look up the output with `get($type->name, $mode, $target)`.
  - No output: record `MissingOutput` on the block; the block yields its child passes joined.
  - Otherwise call `render(RenderInput)`. An output that throws records `OutputFailed` naming the exception class and yields `''`; a non-string result is impossible by type.
  - **Stage 7.** For a block that was drawn (its output was called and did not throw), add its type's `assets`, then each `CollectStepInterface`'s assets (a step that throws, or returns non-strings, records `StepFailed` and contributes nothing). Assets are de-duplicated keeping first appearance.
- `RenderInput::$admin` is the resolved block's `admin` in admin mode and null in front mode. `RenderInput::$context` is the context the block received.
- The result's `output` is the roots' output concatenated. `issues` is the resolved document's issues followed by the render issues in tree order.
- Nothing is cached between calls.
- **Decorator view schemas (deferred from part 1a).** In `ViewStage`, after the view steps have run, for each decorator that applies to the type and declares a `viewSchema`: when the view has an extension under that namespace that fails the schema, record one `InvalidView` issue per failing rule (path prefixed `@namespace:`) and remove that extension from the view. The block still has its view.

- [ ] **Step 1: Write the failing tests**

```php
// tests/Pest.php helpers (guarded): renderer(?callable $setup = null): array{0: DocumentRendererInterface, 1: BlockServiceContainer}
//   => container with MemoryServiceProvider and AppServiceProvider; $setup receives the container.
// outputs(BlockServiceContainer $c): BlockOutputRegistryInterface
// tag(string $element): BlockOutputInterface => anonymous output returning
//   "<{$element}{$input->escaper->attributes($input->attributes)}>{$input->escaper->text((string) ($input->view->values['text'] ?? ''))}{$input->children()}</{$element}>"

test('draws a tree with escaped values and children inside their parent', function () {
    [$renderer] = renderer(function ($c) {
        types($c)->add(type('acme/box', ['children' => true, 'settings' => ['type' => 'object', 'properties' => ['text' => ['type' => 'string']]]]));
        outputs($c)->set(BlockName::fromString('acme/box'), tag('div'));
    });
    $result = $renderer->render(doc(block('a', 'acme/box', ['text' => '<b>hi</b>'], block('b', 'acme/box', ['text' => 'in']))), new RenderRequest());
    expect($result->output)->toBe('<div>&lt;b&gt;hi&lt;/b&gt;<div>in</div></div>')->and($result->hasIssues())->toBeFalse();
});
test('a type with no output passes its children through and reports it', function () {
    [$renderer] = renderer(function ($c) {
        types($c)->add(type('acme/wrap', ['children' => true]));
        types($c)->add(type('acme/box'));
        outputs($c)->set(BlockName::fromString('acme/box'), tag('p'));
    });
    $result = $renderer->render(doc(block('w', 'acme/wrap', [], block('b', 'acme/box'))), new RenderRequest());
    expect($result->output)->toBe('<p></p>')->and(kinds($result->issues))->toBe([IssueKind::MissingOutput])->and($result->issues[0]->blockId)->toBe('w');
});
test('admin mode uses the admin output when there is one and the front output otherwise', function () {
    [$renderer] = renderer(function ($c) {
        types($c)->add(type('acme/a'));
        types($c)->add(type('acme/b'));
        outputs($c)->set(BlockName::fromString('acme/a'), fixedOutput('[a front]'));
        outputs($c)->set(BlockName::fromString('acme/a'), fixedOutput('[a admin]'), Mode::Admin);
        outputs($c)->set(BlockName::fromString('acme/b'), fixedOutput('[b front]'));
    });
    $document = doc(block('1', 'acme/a'), block('2', 'acme/b'));
    expect($renderer->render($document, new RenderRequest(Mode::Admin))->output)->toBe('[a admin][b front]')
        ->and($renderer->render($document, new RenderRequest(Mode::Front))->output)->toBe('[a front][b front]');
});
test('an output that throws blanks only its own block', function () {
    [$renderer] = renderer(function ($c) {
        types($c)->add(type('acme/bad'));
        types($c)->add(type('acme/ok'));
        outputs($c)->set(BlockName::fromString('acme/bad'), new class implements BlockOutputInterface {
            public function render(RenderInput $input): string { throw new RuntimeException('boom'); }
        });
        outputs($c)->set(BlockName::fromString('acme/ok'), fixedOutput('ok'));
    });
    $result = $renderer->render(doc(block('1', 'acme/bad'), block('2', 'acme/ok')), new RenderRequest());
    expect($result->output)->toBe('ok')->and(kinds($result->issues))->toBe([IssueKind::OutputFailed]);
});
test('a block that did not resolve draws nothing and adds no render issue', function () {
    [$renderer] = renderer(function ($c) { types($c)->add(type('acme/ok')); outputs($c)->set(BlockName::fromString('acme/ok'), fixedOutput('ok')); });
    $result = $renderer->render(doc(block('1', 'gone/away'), block('2', 'acme/ok')), new RenderRequest());
    expect($result->output)->toBe('ok')->and(kinds($result->issues))->toBe([IssueKind::UnknownType]);
});
test('render steps add attributes, and a rejected attribute is reported without breaking the block', function () {
    [$renderer] = renderer(function ($c) {
        types($c)->add(type('acme/box'));
        outputs($c)->set(BlockName::fromString('acme/box'), tag('div'));
        steps($c)->add(RenderStepInterface::class, renderStep(fn (AttributeBag $bag) => $bag->withClass('is-animated')->withAttribute('data-fx', 'fade')));
        steps($c)->add(RenderStepInterface::class, renderStep(fn (AttributeBag $bag) => $bag->withAttribute('onclick', 'x()')));
    });
    $result = $renderer->render(doc(block('a', 'acme/box')), new RenderRequest());
    expect($result->output)->toBe('<div class="is-animated" data-fx="fade"></div>')->and(kinds($result->issues))->toBe([IssueKind::InvalidAttribute]);
});
test('each child pass is rendered separately and in order', function () {
    [$renderer] = renderer(function ($c) {
        types($c)->add(type('acme/loop', ['children' => true]));
        types($c)->add(type('acme/item'));
        behaviours($c)->set(BlockName::fromString('acme/loop'), new class implements BlockBehaviourInterface {
            public function view(Block $block, BlockType $type, array $settings, Context $context, Mode $mode): BlockView
            {
                return new BlockView([], childPasses: [new CurrentItem('x', 0, 'q'), new CurrentItem('y', 1, 'q')]);
            }
        });
        outputs($c)->set(BlockName::fromString('acme/loop'), new class implements BlockOutputInterface {
            public function render(RenderInput $input): string { return '<ul>' . implode('|', $input->childPasses) . '</ul>'; }
        });
        outputs($c)->set(BlockName::fromString('acme/item'), new class implements BlockOutputInterface {
            public function render(RenderInput $input): string { return '<li>' . $input->escaper->text((string) $input->context->currentItem()->item) . '</li>'; }
        });
    });
    expect($renderer->render(doc(block('l', 'acme/loop', [], block('i', 'acme/item'))), new RenderRequest())->output)->toBe('<ul><li>x</li>|<li>y</li></ul>');
});
test('assets are collected once, in order, only for blocks that were drawn', function () {
    [$renderer] = renderer(function ($c) {
        types($c)->add(type('acme/a', ['assets' => ['a-style', 'shared']]));
        types($c)->add(type('acme/b', ['assets' => ['shared', 'b-script']]));
        types($c)->add(type('acme/none', ['assets' => ['never']]));
        outputs($c)->set(BlockName::fromString('acme/a'), fixedOutput('a'));
        outputs($c)->set(BlockName::fromString('acme/b'), fixedOutput('b'));
        steps($c)->add(CollectStepInterface::class, collectStep(['from-step']));
    });
    $result = $renderer->render(doc(block('1', 'acme/a'), block('2', 'acme/b'), block('3', 'acme/none')), new RenderRequest());
    expect($result->assets)->toBe(['a-style', 'shared', 'from-step', 'b-script']);
});
test('the admin context reaches the output in admin mode only', function () {
    $seen = [];
    [$renderer] = renderer(function ($c) use (&$seen) {
        types($c)->add(type('acme/box'));
        outputs($c)->set(BlockName::fromString('acme/box'), spyOutput($seen));
    });
    $renderer->render(doc(block('a', 'acme/box')), new RenderRequest(Mode::Front));
    $renderer->render(doc(block('a', 'acme/box')), new RenderRequest(Mode::Admin));
    expect($seen[0]->admin)->toBeNull()->and($seen[1]->admin)->toBeInstanceOf(AdminContext::class);
});
test('an unknown target is a configuration error, and a mode mismatch is rejected', function () {
    [$renderer, $c] = renderer(fn ($c) => types($c)->add(type('acme/box')));
    $resolved = $c->get(DocumentResolverInterface::class)->resolve(doc(block('a', 'acme/box')), Mode::Front);
    expect(fn () => $renderer->render(doc(block('a', 'acme/box')), new RenderRequest(Mode::Front, 'pdf')))->toThrow(UnsupportedTargetException::class)
        ->and(fn () => $renderer->renderResolved($resolved, new RenderRequest(Mode::Admin)))->toThrow(InvalidArgumentException::class);
});
// ViewStageTest addition
test('a decorator extension that fails its view schema is dropped and reported', function () {
    [$resolver] = resolver(function ($c) {
        types($c)->add(type('acme/box'));
        $c->get(DecoratorRegistryInterface::class)->add(new Decorator('acme/fx', ['type' => 'object'], null,
            ['type' => 'object', 'required' => ['class'], 'properties' => ['class' => ['type' => 'string']]]));
        steps($c)->add(ViewStepInterface::class, viewStep(['acme/fx' => ['class' => 5], 'acme/other' => ['kept' => true]]));
    });
    $block = $resolver->resolve(doc(block('a', 'acme/box')), Mode::Front)->find('a');
    expect($block->view->extensions)->toBe(['acme/other' => ['kept' => true]])
        ->and($resolver->resolve(doc(block('a', 'acme/box')), Mode::Front)->issues[0]->kind)->toBe(IssueKind::InvalidView);
});
```

Helpers `fixedOutput(string)`, `renderStep(callable)`, `collectStep(array)`, `spyOutput(array &$seen)` and `viewStep(array $extensions)` go in `tests/Pest.php`, guarded by `function_exists`.

- [ ] **Step 2: Run `vendor/bin/pest --colors=never` in `block-app`.** Expected: FAIL, classes not found.
- [ ] **Step 3: Implement** the renderer, the provider bindings and the view-stage change.
- [ ] **Step 4: Run `vendor/bin/pest --colors=never` in `block-domain`, `block-app` and `block-infra-memory`.** Expected: PASS in each.
- [ ] **Step 5: Commit** `feat(block-app): add the document renderer`.

---

### Task 4: The block package entry point

**Files:**
- Create in `block-domain/src/Contracts/`: `BlockPackageInterface.php`, `BlockPackageRegistrarInterface.php`, `BlockPackageLoaderInterface.php`; `src/Exceptions/BlockPackageException.php`
- Create in `block-app/src/Packages/`: `BlockPackageRegistrar.php`, `BlockPackageLoader.php`
- Modify: `block-app/src/AppServiceProvider.php` (bind `BlockPackageLoaderInterface`)
- Test: `block-app/tests/Packages/BlockPackageLoaderTest.php`

**Interfaces:**
- `BlockPackageInterface`: `name(): string` (a lower-case `vendor/package` string, same rule as a block name), `register(BlockPackageRegistrarInterface $registrar): void`.
- `BlockPackageRegistrarInterface` — the only thing a block package touches:
  - `type(BlockType $type): void`
  - `behaviour(BlockName $type, BlockBehaviourInterface $behaviour): void`
  - `output(BlockName $type, BlockOutputInterface $output, Mode $mode = Mode::Front, string $target = 'html'): void`
  - `decorator(Decorator $decorator): void`
  - `migration(BlockName $type, SettingsMigrationInterface $migration): void`
  - `dataSource(DataSourceInterface $source): void`
  - `step(string $stepInterface, object $step, int $priority = 10): void`
- `BlockPackageLoaderInterface`: `load(BlockPackageInterface $package): void`, `loaded(): array` (list of package names, in load order), `typesOf(string $packageName): array` (list of `BlockName` the package registered, in order).
- `BlockPackageLoader::__construct(...)` takes the seven registries it writes to (types, behaviours, outputs, decorators, migrations, data sources, steps). `load()`:
  - throws `BlockPackageException` (extends `\RuntimeException`) for an invalid package name or a package name already loaded;
  - gives the package a `BlockPackageRegistrar` that records what it registers;
  - throws `BlockPackageException` when the package registers a type whose name's vendor differs from the package name's vendor, or a type name another loaded package already registered;
  - registration failures from the registries (for example `UnsupportedSchemaException`) propagate unchanged;
  - a package that registers nothing loads without error.
- A behaviour, output or migration for a type the package did not itself register is allowed (a theme package may replace one block's output).

- [ ] **Step 1: Write the failing tests**

```php
// helper in the test file: package(string $name, callable $register): BlockPackageInterface => anonymous class
test('a package registers its types, behaviours and outputs through the registrar', function () {
    [$renderer, $c] = renderer();
    $c->get(BlockPackageLoaderInterface::class)->load(package('acme/basic', function (BlockPackageRegistrarInterface $r) {
        $r->type(type('acme/note', ['settings' => ['type' => 'object', 'properties' => ['text' => ['type' => 'string']]]]));
        $r->output(BlockName::fromString('acme/note'), tag('p'));
    }));
    $loader = $c->get(BlockPackageLoaderInterface::class);
    expect($renderer->render(doc(block('a', 'acme/note', ['text' => 'hi'])), new RenderRequest())->output)->toBe('<p>hi</p>')
        ->and($loader->loaded())->toBe(['acme/basic'])
        ->and(array_map(fn (BlockName $n) => $n->toString(), $loader->typesOf('acme/basic')))->toBe(['acme/note']);
});
test('a package cannot register a type under another vendor or reuse a loaded type name', function () {
    [, $c] = renderer();
    $loader = $c->get(BlockPackageLoaderInterface::class);
    expect(fn () => $loader->load(package('acme/basic', fn ($r) => $r->type(type('other/note')))))->toThrow(BlockPackageException::class);
    $loader->load(package('acme/one', fn ($r) => $r->type(type('acme/note'))));
    expect(fn () => $loader->load(package('acme/two', fn ($r) => $r->type(type('acme/note')))))->toThrow(BlockPackageException::class);
});
test('a package cannot be loaded twice and needs a valid name', function () {
    [, $c] = renderer();
    $loader = $c->get(BlockPackageLoaderInterface::class);
    $loader->load(package('acme/basic', fn ($r) => null));
    expect(fn () => $loader->load(package('acme/basic', fn ($r) => null)))->toThrow(BlockPackageException::class)
        ->and(fn () => $loader->load(package('Not A Name', fn ($r) => null)))->toThrow(BlockPackageException::class);
});
test('another package may replace the output of a block it did not add', function () {
    [$renderer, $c] = renderer();
    $loader = $c->get(BlockPackageLoaderInterface::class);
    $loader->load(package('acme/basic', function ($r) { $r->type(type('acme/note')); $r->output(BlockName::fromString('acme/note'), fixedOutput('plain')); }));
    $loader->load(package('theme/fancy', fn ($r) => $r->output(BlockName::fromString('acme/note'), fixedOutput('fancy'))));
    expect($renderer->render(doc(block('a', 'acme/note')), new RenderRequest())->output)->toBe('fancy');
});
test('a package can add decorators, data sources, migrations, admin outputs and steps', function () {
    [$renderer, $c] = renderer();
    $c->get(BlockPackageLoaderInterface::class)->load(package('acme/full', function (BlockPackageRegistrarInterface $r) {
        $name = BlockName::fromString('acme/note');
        $r->type(type('acme/note', ['version' => 2, 'settings' => ['type' => 'object', 'properties' => ['text' => ['type' => 'string']]]]));
        $r->migration($name, migrationTo(['text' => 'migrated']));
        $r->output($name, tag('p'));
        $r->output($name, fixedOutput('[admin]'), Mode::Admin);
        $r->decorator(new Decorator('acme/fx', ['type' => 'object']));
        $r->dataSource(new ArrayDataSource('acme/values', ['k' => 'bound']));
        $r->step(RenderStepInterface::class, renderStep(fn (AttributeBag $bag) => $bag->withClass('fx')));
    }));
    $old = block('a', 'acme/note', ['title' => 'x']);
    expect($renderer->render(doc($old), new RenderRequest())->output)->toBe('<p class="fx">migrated</p>')
        ->and($renderer->render(doc($old), new RenderRequest(Mode::Admin))->output)->toBe('[admin]')
        ->and($c->get(DataSourceRegistryInterface::class)->get('acme/values'))->not->toBeNull()
        ->and($c->get(DecoratorRegistryInterface::class)->get('acme/fx'))->not->toBeNull();
});
```

`migrationTo(array $settings)` is a `tests/Pest.php` helper returning an anonymous `SettingsMigrationInterface` with `from()` 1 whose `migrate()` returns `$settings`.

- [ ] **Step 2: Run `vendor/bin/pest --colors=never` in `block-app`.** Expected: FAIL, classes not found.
- [ ] **Step 3: Implement** the contracts, registrar, loader and binding.
- [ ] **Step 4: Run `vendor/bin/pest --colors=never` in `block-domain` and `block-app`.** Expected: PASS.
- [ ] **Step 5: Commit** `feat(block): add the block package entry point`.

---

### Task 5: The block package suite and an example package

**Files:**
- Create: `block-contract-tests/src/blockPackageContract.php`; modify `block-contract-tests/composer.json` (add `speedyspec/block-app` and `speedyspec/block-infra-memory` to `require`; add the file to `autoload.files`)
- Create: `packages/block-example/{composer.json,phpunit.xml,README.md,tests/Pest.php}`, `src/ExampleBlockPackage.php`, `src/Notice/{NoticeOutput,NoticeAdminOutput}.php`, `src/Listing/{ListBehaviour,ListOutput,ListItemOutput}.php`
- Test: `block-example/tests/{PackageContractTest,OutputTest}.php`
- Modify: the repository `README.md` (list the packages; point to `block-example` as the model for writing a block package)

**Interfaces:**
- `blockPackageContract(Closure $makePackage, string $composerJsonPath, array $with = []): void` — `$makePackage` returns a fresh `BlockPackageInterface`; `$with` is a list of closures returning other packages to load first (for parent types the package under test does not define). It registers a `describe('BlockPackageInterface contract', ...)` block. For each test it builds a fresh container with `MemoryServiceProvider` and `AppServiceProvider`, loads `$with` then the package, and asserts:
  1. **Dependencies.** The `require` section of the JSON at `$composerJsonPath` contains only `php`, entries starting `ext-`, and `speedyspec/block-domain`.
  2. **Shape.** The package registers at least one block type. Every type it registers has a non-empty catalogue title, at least one example, and a front `html` output (`has($name, Mode::Front)`).
  3. **Examples resolve and draw.** For every example of every type, in both modes: build a one-block document (see placement below), render it, and expect no issues and a string output.
  4. **Escaping.** For every type and every top-level setting whose schema is `type: string` with no `enum`, `const` or `pattern`: take the first example, set that setting to `<script>alert("x")</script>`, skip the case if the result no longer validates against the settings schema, render in front mode, and expect the output not to contain `<script>alert(`.
  5. **Private settings stay out of front output.** For every type and every private top-level string setting: take the first example, set the setting to `private-marker-<random>`, skip if it no longer validates, render in front mode, and expect the output not to contain the marker.
  6. **Admin context stays out of front output.** For every type: resolve the first example in admin mode; for every string of 8 or more characters among the behaviour-supplied `admin->values` (recursively), expect the front-mode output not to contain it.
- **Placement in the suite.** A type whose parent policy is `any` is placed at the root. Otherwise the block is placed inside a block of its first allowed parent type, built from that parent type's first example, and the parent is placed by the same rule (at most 5 levels; a parent type that is not registered fails the test with a message naming it). The assertions in 3 to 6 apply to the whole rendered output.
- `block-example` (`speedyspec/block-example`, package name `example/blocks`), registered by `ExampleBlockPackage implements BlockPackageInterface`:
  - **`example/notice`**: settings `text` (string, default `''`), `tone` (enum `info`, `warning`; default `info`), `internalNote` (string, `x-private`, default `''`); one example `['text' => 'Heads up', 'tone' => 'warning', 'internalNote' => 'Reviewed by legal']`; assets `['example-notice']`. Front output: `<div class="notice notice--{tone}">{text}</div>`, with the attribute bag applied to the `div` and merged with those classes. Admin output: the same `div`, followed by `<p class="notice__note">{internalNote}</p>` when the note is not empty.
  - **`example/list`**: children policy `['example/list-item']`; settings `ordered` (boolean, default false), `label` (string, default `''`); provides `['example/list-ordered' => 'ordered']`; a behaviour `ListBehaviour` whose view has `ordered`, `label` and `count` (the number of the block's children); one example `['label' => 'Steps']`. Output: `<ol>` or `<ul>` with `aria-label="{label}"` when the label is not empty, containing the children.
  - **`example/list-item`**: parent policy `['example/list']`; uses `['example/list-ordered']`; settings `text` (string, default `''`); one example `['text' => 'First']`. Output: `<li>{text}</li>`.
  - Every value is written through `$input->escaper`; every root element applies `$input->escaper->attributes(...)`.
- `block-example/composer.json`: `require` only `php` and `speedyspec/block-domain`; `require-dev` Pest, `speedyspec/block-contract-tests`, `speedyspec/block-app`, `speedyspec/block-infra-memory`.
- `block-example/README.md`: how a block package is laid out, what it may depend on, and how to run the shared suite, in under 60 lines.

- [ ] **Step 1: Write the failing tests**

```php
// block-example/tests/PackageContractTest.php
describe('ExampleBlockPackage', function () {
    blockPackageContract(fn () => new ExampleBlockPackage(), __DIR__ . '/../composer.json');
});

// block-example/tests/OutputTest.php — helper html(Block $root, Mode $mode = Mode::Front): string renders one document
// with MemoryServiceProvider, AppServiceProvider and ExampleBlockPackage loaded.
test('a notice draws its tone and escaped text', function () {
    expect(html(block('n', 'example/notice', ['text' => 'Mind the <gap>', 'tone' => 'warning'])))
        ->toBe('<div class="notice notice--warning">Mind the &lt;gap&gt;</div>');
});
test('the internal note appears in the admin output only', function () {
    $notice = block('n', 'example/notice', ['text' => 'Hi', 'internalNote' => 'Reviewed by legal']);
    expect(html($notice, Mode::Front))->not->toContain('Reviewed by legal')
        ->and(html($notice, Mode::Admin))->toBe('<div class="notice notice--info">Hi</div><p class="notice__note">Reviewed by legal</p>');
});
test('a list draws its items in order and switches element when ordered', function () {
    $items = [block('a', 'example/list-item', ['text' => 'One']), block('b', 'example/list-item', ['text' => 'Two'])];
    expect(html(block('l', 'example/list', ['label' => 'Steps'], ...$items)))->toBe('<ul aria-label="Steps"><li>One</li><li>Two</li></ul>')
        ->and(html(block('l', 'example/list', ['ordered' => true], ...$items)))->toBe('<ol><li>One</li><li>Two</li></ol>');
});
test('a list item outside a list is not drawn', function () {
    expect(html(block('a', 'example/list-item', ['text' => 'Loose'])))->toBe('');
});
```

In `block-app/tests/Packages/`, add `BlockPackageContractFailureTest.php` proving the suite's checks have teeth without running failing tests: call the suite's underlying check helpers (expose them from `blockPackageContract.php` as plain functions in the `SpeedySpec\Block\ContractTests` namespace: `packageRuntimeRequirements(string $composerJsonPath): array`, `unescapedMarkupFindings(...)`, `privateLeakFindings(...)`) against a deliberately wrong fixture package (an output that prints a setting unescaped and prints a private setting in front mode; a `composer.json` fixture requiring `speedyspec/block-app`), and assert each helper reports the defect.

- [ ] **Step 2: Scaffold `block-example`,** update `block-contract-tests/composer.json`, run `composer update` in `block-example` and `composer update speedyspec/block-contract-tests` in `block-app` and `block-infra-memory`, then `vendor/bin/pest --colors=never` in `block-example`. Expected: FAIL, classes not found. `block-contract-tests` now requires `block-app` while `block-app` dev-requires `block-contract-tests`; Composer satisfies the former with the root package. If it cannot resolve, stop and report Composer's output; do not weaken any `require` section.
- [ ] **Step 3: Implement** the suite, its check helpers, the example package and the READMEs.
- [ ] **Step 4: Run `vendor/bin/pest --colors=never` in `block-domain`, `block-app`, `block-infra-memory` and `block-example`.** Expected: PASS in each.
- [ ] **Step 5: Commit** `feat(block): add the block package suite and an example package`.

---

## Out of scope for this plan

Builder contracts (catalogue, placement, editing operations, preview, block templates: part 1c); output targets other than HTML; client-side behaviour; CSS generation, responsive values and design tokens (phase 3); conditions and loops beyond the existing child-pass mechanism (phase 4); structural blocks (phase 5); the WordPress adapter; forms. The remaining items in the design document's "Deferred from part 1a" list stay deferred, except the decorator view schema enforcement taken in Task 3.
