# Block System Part 1a (model and resolving) Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Ship a WordPress-independent block model that can define block types, build and store documents of blocks, and resolve a document to each block's final settings, view and context in front or admin mode, without drawing anything.

**Architecture:** A new monorepo, `packages/speedyspec-api-block/`, with Domain-Driven Design layer packages under `packages/`: `block-domain` (every contract plus value objects and a per-contract service container), `block-app` (framework-free defaults: schema validator, registries, serializer, client export, the five resolving stages), `block-infra-memory` (in-memory repository and data source) and `block-contract-tests` (shared Pest contract suites). Output, the block package entry point and builder contracts are parts 1b and 1c and are out of scope here.

**Tech Stack:** PHP 8.4, Composer path repositories, Pest 3/4.

**Spec:** `docs/superpowers/specs/2026-11-17-block-model-design.md` (in the `wordpress-develop` checkout)

## Global Constraints

- Work only inside `/Users/jacobsantos/repos/respite-studios/wordpress-develop/packages/speedyspec-api-block/`. It is a clone of `SpeedySpec/speedyspec-api-block`, its own git repository. Never run git commands against the surrounding `wordpress-develop` repository and never change files outside this directory. Never push; pushing is the project owner's decision at the end.
- Branch `feat/block-phase-1a`, cut from `main` (which already holds the repository's initial commit with a one-line `README.md`). One commit per task. Never commit `vendor/`, `composer.lock` or `.phpunit.cache/`.
- PHP `>=8.4`; every PHP file starts with `declare(strict_types=1);`; 4-space indentation.
- Braces: the opening brace of every class, interface, enum, function, method and control structure (`if`, `else`, `foreach`, `for`, `while`, `switch`, `try`, `catch`, `match` is exempt) goes on its own line. Closures, arrow functions and anonymous classes keep their brace on the same line.
- Docblocks follow `/Users/jacobsantos/repos/respite-studios/wordpress-develop/.claude/rules/coding-style.md`: a summary line, the "why" not the "how", no `@return void`, `@since 0.1.0`, lines at most 120 characters.
- `block-domain` requires only PHP, calls nothing outside PHP's standard library, and has no `final` classes. No package may reference a WordPress function, class or package.
- `block-app` and `block-infra-memory` require only `speedyspec/block-domain` at runtime. `block-contract-tests` requires `speedyspec/block-domain`, `speedyspec/block-app`, `speedyspec/block-infra-memory` and Pest.
- Namespaces: `SpeedySpec\Block\Domain\`, `SpeedySpec\Block\App\`, `SpeedySpec\Block\Infra\Memory\`, `SpeedySpec\Block\ContractTests\`. Test namespaces add `\Tests`.
- Composer names `speedyspec/block-domain`, `speedyspec/block-app`, `speedyspec/block-infra-memory`, `speedyspec/block-contract-tests`; each `"version": "0.1.0"`, license `GPL-2.0-or-later`, with the same `archive.exclude`, `config.allow-plugins`, `minimum-stability`, `prefer-stable` and `phpunit.xml` as `packages/speedyspec-api-seo/packages/wp-seo-app`. Siblings are linked with `"repositories": [{"type": "path", "url": "../*"}]` and required as `"^0.1"`. `block-app` has `"provide": {"speedyspec/block-app-implementation": "1.0"}`.
- Tests are Pest, files end `Test.php`, run with `composer install && vendor/bin/pest --colors=never` inside the package directory. Output must be free of warnings and deprecations.
- Schemas are plain PHP arrays in JSON Schema form. Supported keywords, exactly: `type` (`string`, `integer`, `number`, `boolean`, `array`, `object`, `null`, or a list of these), `enum`, `const`, `default`, `required`, `properties`, `additionalProperties`, `items`, `minLength`, `maxLength`, `pattern`, `minimum`, `maximum`, `minItems`, `maxItems`, and the annotations `title`, `description`, `x-private`, `x-control`, `x-group`. Any other keyword makes a schema unsupported.
- `x-private` is honoured on top-level properties of a settings schema only. A settings schema with `x-private` anywhere deeper is unsupported.
- Resolving never throws for bad content; it records an `Issue` and continues with the rest of the tree.

## Review Focus

Each line has a pinning test in the task named.

1. **A private setting reaching a visitor.** Expected: absent from a front-mode view, absent from a front-mode client export, and never overwritten by a binding. Tasks 6 and 8.
2. **A stored document that names a block type nobody registered.** Expected: it loads, saves back byte-for-byte equivalent, resolves with an issue for that block only, and a front-mode export strips all of that block's settings. Tasks 6 and 7.
3. **A data source returning a value the setting's schema rejects** (a visitor's over-long resubmitted value). Expected: the block is still resolved using the fallback, stored value or default, with an issue recorded. Task 7.
4. **Two blocks with the same id in one tree.** Expected: constructing the tree throws; duplicating a block gives every copy a new id. Tasks 3 and 4.
5. **A schema using a keyword the validator does not implement** (for example `oneOf` or `format`). Expected: registering the block type throws, so the rule is never silently ignored. Tasks 2 and 4.

---

## File Structure

```
packages/speedyspec-api-block/
├── README.md, .gitignore
└── packages/
    ├── block-domain/src/
    │   ├── BlockServiceContainer.php
    │   ├── Contracts/        one interface per file (listed per task)
    │   ├── ValueObjects/     BlockName, Mode, Issue, IssueKind, ValidationResult, ValidationError, ChildrenPolicy,
    │   │                     ParentPolicy, CatalogueMetadata, BlockType, SettingPath, Binding, Decorator, Context,
    │   │                     CurrentItem, DataSourceResult, BlockView, AdminContext, ResolvedBlock, ResolvedDocument
    │   ├── Entities/         Block, BlockTree, Document
    │   └── Exceptions/       one per file
    ├── block-app/src/
    │   ├── Schema/SubsetSchemaValidator.php
    │   ├── Registries/       BlockTypeRegistry, BlockBehaviourRegistry, DecoratorRegistry, DataSourceRegistry,
    │   │                     SettingsMigrationRegistry, StepRegistry
    │   ├── Ids/              RandomBlockIdGenerator, BlockDuplicator
    │   ├── Serialization/    JsonDocumentSerializer, ClientExport
    │   ├── Resolving/        PrepareStage, GateStage, ResolveStage, ValidateStage, ViewStage, DocumentResolver
    │   └── AppServiceProvider.php
    ├── block-infra-memory/src/   InMemoryDocumentRepository, ArrayDataSource, MemoryServiceProvider
    └── block-contract-tests/src/ schemaValidatorContract.php, documentRepositoryContract.php,
                                  dataSourceContract.php, documentSerializerContract.php
```

---

### Task 1: Repository, domain package and service container

**Files:**
- Modify: `README.md` (already exists with only a title); Create: `.gitignore` (`vendor/`, `composer.lock`, `.phpunit.cache/`, `.DS_Store`).
- Create: `packages/block-domain/{composer.json,phpunit.xml,tests/Pest.php}`
- Create: `packages/block-domain/src/BlockServiceContainer.php`, `src/Exceptions/{ServiceNotRegisteredException,ServiceAlreadyResolvedException,InvalidBlockNameException}.php`, `src/ValueObjects/{BlockName,Mode}.php`
- Test: `tests/BlockServiceContainerTest.php`, `tests/ValueObjects/BlockNameTest.php`

**Interfaces:**
- Produces `BlockServiceContainer` (not `final`, public constructor): `add(string $contract, callable $factory): static` (factory receives the container), `remove(string $contract): static`, `extend(string $contract, callable $decorator): static` (decorator receives the inner instance and the container; decorators stack in the order added, first added innermost), `has(string $contract): bool`, `get(string $contract): object`, `static shared(): self`, `static resetShared(): void`. `get()` is lazy and caches. `add()`, `remove()` and `extend()` throw `ServiceAlreadyResolvedException` once that contract has been resolved; `get()` and `extend()` throw `ServiceNotRegisteredException` when nothing is bound; `add()` and `remove()` discard pending decorators. Both exceptions extend `\RuntimeException`. Behaviour matches `packages/speedyspec-api-seo/packages/wp-seo-domain/src/SeoServiceContainer.php`; read it, but write this one in this repository's brace style.
- Produces `BlockName`: `static fromString(string $value): self` (must match `/^[a-z][a-z0-9-]*\/[a-z][a-z0-9-]*\z/`, else `InvalidBlockNameException`), `toString(): string`, `vendor(): string`, `name(): string`, `equals(self $other): bool`.
- Produces `enum Mode: string { case Front = 'front'; case Admin = 'admin'; }`.

- [ ] **Step 1: Prepare the repository.** In `packages/speedyspec-api-block/` (already cloned, on `main`), run `git switch -c feat/block-phase-1a`. Extend `README.md` with one paragraph (what the block system is, that packages live under `packages/`, how to run a package's tests) and add `.gitignore`; these are committed with this task.
- [ ] **Step 2: Scaffold `block-domain`** and run `composer install`. Expected: Pest installed.
- [ ] **Step 3: Write the failing tests**

```php
// tests/ValueObjects/BlockNameTest.php
test('accepts vendor/name and exposes its parts', function () {
    $name = BlockName::fromString('speedyspec/text-field');
    expect($name->vendor())->toBe('speedyspec')->and($name->name())->toBe('text-field')
        ->and($name->toString())->toBe('speedyspec/text-field');
});
test('rejects names that are not lower-case vendor/name', function (string $value) {
    expect(fn () => BlockName::fromString($value))->toThrow(InvalidBlockNameException::class);
})->with(['paragraph', 'Core/Paragraph', 'core/', '/x', 'a/b/c', 'core/para graph', "core/x\n", '']);

// tests/BlockServiceContainerTest.php  (afterEach: BlockServiceContainer::resetShared())
test('get resolves lazily, passes the container, and caches', function () {
    $c = new BlockServiceContainer();
    $calls = 0;
    $c->add('x', function (BlockServiceContainer $inner) use (&$calls, $c) {
        $calls++;
        expect($inner)->toBe($c);
        return new stdClass();
    });
    expect($calls)->toBe(0)->and($c->get('x'))->toBe($c->get('x'))->and($calls)->toBe(1);
});
test('a later add replaces an unresolved binding and drops its decorators', function () {
    $c = (new BlockServiceContainer())->add('x', fn () => new ArrayObject(['first']));
    $c->extend('x', fn (ArrayObject $inner) => new ArrayObject([$inner[0], 'decorated']));
    $c->add('x', fn () => new ArrayObject(['second']));
    expect($c->get('x')->getArrayCopy())->toBe(['second']);
});
test('decorators stack with the first added innermost', function () {
    $c = (new BlockServiceContainer())->add('x', fn () => new ArrayObject(['inner']));
    $c->extend('x', fn (ArrayObject $i) => new ArrayObject([...$i->getArrayCopy(), 'a']));
    $c->extend('x', fn (ArrayObject $i) => new ArrayObject([...$i->getArrayCopy(), 'b']));
    expect($c->get('x')->getArrayCopy())->toBe(['inner', 'a', 'b']);
});
test('add, remove and extend throw once the contract was resolved', function () {
    $c = (new BlockServiceContainer())->add('x', fn () => new stdClass());
    $c->get('x');
    expect(fn () => $c->add('x', fn () => new stdClass()))->toThrow(ServiceAlreadyResolvedException::class)
        ->and(fn () => $c->remove('x'))->toThrow(ServiceAlreadyResolvedException::class)
        ->and(fn () => $c->extend('x', fn ($i) => $i))->toThrow(ServiceAlreadyResolvedException::class);
});
test('get and extend throw for an unbound contract and has reports it', function () {
    $c = new BlockServiceContainer();
    expect($c->has('x'))->toBeFalse()
        ->and(fn () => $c->get('x'))->toThrow(ServiceNotRegisteredException::class)
        ->and(fn () => $c->extend('x', fn ($i) => $i))->toThrow(ServiceNotRegisteredException::class);
});
test('shared returns one instance until reset', function () {
    $a = BlockServiceContainer::shared();
    expect(BlockServiceContainer::shared())->toBe($a);
    BlockServiceContainer::resetShared();
    expect(BlockServiceContainer::shared())->not->toBe($a);
});
```

- [ ] **Step 4: Run `vendor/bin/pest --colors=never`.** Expected: FAIL, classes not found.
- [ ] **Step 5: Implement** the container, exceptions, `BlockName` and `Mode`.
- [ ] **Step 6: Run `vendor/bin/pest --colors=never`.** Expected: PASS.
- [ ] **Step 7: Commit** `feat(block-domain): add service container and block name`.

---

### Task 2: Schema validation

**Files:**
- Create in `block-domain/src/`: `ValueObjects/{ValidationResult,ValidationError}.php`, `Contracts/SchemaValidatorInterface.php`, `Exceptions/UnsupportedSchemaException.php`
- Create: `packages/block-app/{composer.json,phpunit.xml,tests/Pest.php}`, `src/Schema/SubsetSchemaValidator.php`
- Create: `packages/block-contract-tests/{composer.json,src/schemaValidatorContract.php}` (function files loaded through `autoload.files`; no tests of its own)
- Test: `block-app/tests/Schema/SubsetSchemaValidatorTest.php`

**Interfaces:**
- Produces `ValidationError` (readonly): `__construct(string $path, string $keyword, string $message)`. `$path` is dot-separated from the root (`''` for the root, `items.0.label` for nested).
- Produces `ValidationResult`: `static valid(): self`, `static invalid(ValidationError ...$errors): self`, `isValid(): bool`, `errors(): array` (list of `ValidationError`).
- Produces `SchemaValidatorInterface`:
  - `assertSupported(array $schema, bool $allowPrivate = false): void` — throws `UnsupportedSchemaException` (extends `\InvalidArgumentException`) naming the first unsupported keyword and its path; with `$allowPrivate` true, `x-private` is allowed on top-level properties only.
  - `validate(mixed $value, array $schema): ValidationResult`.
  - `withDefaults(array $value, array $schema): array` — fills missing object properties from `default`, recursively through `properties`; never overwrites a present key.
  - `privateProperties(array $schema): array` — list of top-level property names marked `x-private: true`.
  - `subschema(array $schema, array $segments): ?array` — the schema at a property path (`['style', 'color']`), or null when the path is not declared.
- Produces `SubsetSchemaValidator implements SchemaValidatorInterface` in `SpeedySpec\Block\App\Schema`.
- A `required` failure is reported at the missing property's path. A declared `default` counts even when it is `null` (test with `array_key_exists`). `withDefaults` keeps the value's existing keys in place and appends filled ones in schema order.
- Type rules: `integer` accepts PHP ints only; `number` accepts ints and floats; `array` accepts list arrays; `object` accepts associative arrays and the empty array; an empty array satisfies both `array` and `object`. `pattern` is a PCRE body without delimiters, matched with the `u` flag.
- Produces `SpeedySpec\Block\ContractTests\schemaValidatorContract(Closure $make): void` registering a `describe('SchemaValidatorInterface contract', ...)` block that asserts: a value matching a typed object schema is valid; a missing `required` property is invalid with keyword `required`; `withDefaults` fills a missing property and leaves a present one; an unsupported keyword makes `assertSupported` throw.

- [ ] **Step 1: Write the failing tests**

```php
// block-app/tests/Schema/SubsetSchemaValidatorTest.php
use function SpeedySpec\Block\ContractTests\schemaValidatorContract;

schemaValidatorContract(fn () => new SubsetSchemaValidator());

function schema(): array
{
    return ['type' => 'object', 'required' => ['label'], 'additionalProperties' => false, 'properties' => [
        'label' => ['type' => 'string', 'minLength' => 1, 'maxLength' => 5],
        'size' => ['type' => 'integer', 'minimum' => 1, 'maximum' => 3, 'default' => 2],
        'kind' => ['enum' => ['a', 'b']],
        'tags' => ['type' => 'array', 'minItems' => 1, 'maxItems' => 2, 'items' => ['type' => 'string', 'pattern' => '^[a-z]+$']],
        'style' => ['type' => 'object', 'properties' => ['color' => ['type' => ['string', 'null'], 'default' => null]]],
    ]];
}
test('reports each failing rule with its path and keyword', function (array $value, string $path, string $keyword) {
    $errors = (new SubsetSchemaValidator())->validate($value, schema())->errors();
    expect($errors)->toHaveCount(1)->and($errors[0]->path)->toBe($path)->and($errors[0]->keyword)->toBe($keyword);
})->with([
    [[], 'label', 'required'],
    [['label' => 'toolong'], 'label', 'maxLength'],
    [['label' => ''], 'label', 'minLength'],
    [['label' => 'ok', 'size' => 9], 'size', 'maximum'],
    [['label' => 'ok', 'size' => '2'], 'size', 'type'],
    [['label' => 'ok', 'kind' => 'c'], 'kind', 'enum'],
    [['label' => 'ok', 'tags' => []], 'tags', 'minItems'],
    [['label' => 'ok', 'tags' => ['a', 'B']], 'tags.1', 'pattern'],
    [['label' => 'ok', 'extra' => 1], 'extra', 'additionalProperties'],
    [['label' => 'ok', 'style' => ['color' => 5]], 'style.color', 'type'],
]);
test('a pattern cannot be bypassed with a trailing newline', function () {
    expect((new SubsetSchemaValidator())->validate(['label' => 'ok', 'tags' => ["a\n"]], schema())->isValid())->toBeFalse();
});
test('withDefaults fills nested defaults and keeps present values', function () {
    expect((new SubsetSchemaValidator())->withDefaults(['label' => 'x', 'style' => []], schema()))
        ->toBe(['label' => 'x', 'style' => ['color' => null], 'size' => 2]);
});
test('unsupported keywords are rejected with their path', function (array $schema, string $needle) {
    expect(fn () => (new SubsetSchemaValidator())->assertSupported($schema))
        ->toThrow(UnsupportedSchemaException::class, $needle);
})->with([
    [['oneOf' => []], 'oneOf'],
    [['type' => 'object', 'properties' => ['a' => ['format' => 'email']]], 'properties.a.format'],
    [['type' => 'object', 'properties' => ['a' => ['x-private' => true]]], 'x-private'],
]);
test('x-private is allowed only on top-level properties when asked', function () {
    $v = new SubsetSchemaValidator();
    $top = ['type' => 'object', 'properties' => ['to' => ['type' => 'string', 'x-private' => true], 'b' => ['type' => 'string']]];
    $deep = ['type' => 'object', 'properties' => ['a' => ['type' => 'object', 'properties' => ['b' => ['x-private' => true]]]]];
    $v->assertSupported($top, allowPrivate: true);
    expect($v->privateProperties($top))->toBe(['to'])
        ->and(fn () => $v->assertSupported($deep, allowPrivate: true))->toThrow(UnsupportedSchemaException::class);
});
test('subschema follows a property path', function () {
    $v = new SubsetSchemaValidator();
    expect($v->subschema(schema(), ['style', 'color']))->toBe(['type' => ['string', 'null'], 'default' => null])
        ->and($v->subschema(schema(), ['nope']))->toBeNull();
});
```

- [ ] **Step 2: Scaffold `block-app` and `block-contract-tests`,** run `composer install` in `block-app`, then `vendor/bin/pest --colors=never`. Expected: FAIL, classes not found.
- [ ] **Step 3: Implement** the domain types, the validator and the contract function.
- [ ] **Step 4: Run `vendor/bin/pest --colors=never` in `block-app` and `block-domain`.** Expected: PASS.
- [ ] **Step 5: Commit** `feat(block): add subset JSON Schema validator`.

---

### Task 3: Block types, blocks, trees and documents

**Files:**
- Create in `block-domain/src/ValueObjects/`: `ChildrenPolicy.php`, `ParentPolicy.php`, `CatalogueMetadata.php`, `BlockType.php`, `SettingPath.php`, `Binding.php`, `Decorator.php`
- Create in `block-domain/src/Entities/`: `Block.php`, `BlockTree.php`, `Document.php`
- Create in `block-domain/src/Exceptions/`: `DuplicateBlockIdException.php`, `InvalidSettingPathException.php`, `InvalidBlockTypeException.php`
- Test: `block-domain/tests/ValueObjects/{BlockTypeTest,SettingPathTest,PoliciesTest}.php`, `tests/Entities/BlockTreeTest.php`

**Interfaces (all readonly or immutable):**
- `ChildrenPolicy`: `static none()`, `static any()`, `static only(BlockName ...$names)`, `allows(BlockName $child): bool`, `static fromArray(mixed $value): self` (`false`/`null`/`[]` is none, `true` is any, a list of names is only), `toArray(): bool|array`.
- `ParentPolicy`: `static any()`, `static only(BlockName ...$names)`, `allows(?BlockName $parent): bool` (a null parent is the document root; allowed only by `any()`), `static fromArray(mixed $value): self` (`null`/`true` is any, a list is only), `toArray(): bool|array`.
- `CatalogueMetadata`: `__construct(string $title, string $description = '', string $category = '', array $keywords = [], ?string $icon = null)`; `toArray()`, `static fromArray(array)`.
- `BlockType`: `__construct(BlockName $name, CatalogueMetadata $catalogue, int $version = 1, array $settingsSchema = ['type' => 'object'], ?array $viewSchema = null, ?ChildrenPolicy $children = null, ?ParentPolicy $parents = null, array $provides = [], array $uses = [], array $assets = [], array $examples = [])`. Null policies mean `ChildrenPolicy::none()` and `ParentPolicy::any()`. `$provides` maps a namespaced context key to a view key; `$uses` is a list of context keys; `$examples` is a list of settings arrays. Public readonly properties of the same names. `static fromArray(array $data): self` and `toArray(): array` using keys `name`, `version`, `title`, `description`, `category`, `keywords`, `icon`, `settings`, `view`, `children`, `parents`, `provides`, `uses`, `assets`, `examples`. `fromArray` throws `InvalidBlockTypeException` when `name` or `title` is missing or `version < 1`.
- `SettingPath`: `static fromString(string $path): self`. Syntax: `key.sub` addresses a block setting; `@vendor/decorator:key.sub` addresses a setting of that decorator's extension. `namespace(): ?string`, `segments(): array` (non-empty list), `toString(): string`. Empty segments or a malformed namespace throw `InvalidSettingPathException`.
- `Binding`: `static to(string $source, array $arguments = []): self`, `withFallback(mixed $fallback): self`, readonly `source`, `arguments`, `fallback`, `hasFallback`; `toArray()`, `static fromArray(array)`.
- `Decorator`: `__construct(string $namespace, array $settingsSchema, ?array $appliesTo = null, ?array $viewSchema = null)` (`$namespace` is a `vendor/name` string validated like a block name; `$appliesTo` null means every block type, otherwise a list of block type name strings); `appliesTo(BlockName $type): bool`.
- `Block`: `__construct(string $id, BlockName $type, int $typeVersion = 1, array $settings = [], array $extensions = [], array $bindings = [], array $children = [])` (`$extensions` maps decorator namespace to a settings array; `$bindings` maps a setting-path string to a `Binding`; `$children` is a list of `Block`). Public readonly properties. `withId(string)`, `withTypeVersion(int)`, `withSettings(array)`, `withExtension(string $namespace, array $settings)`, `withBinding(SettingPath $path, Binding $binding)`, `withoutBinding(SettingPath $path)`, `withChildren(Block ...$children)`; each returns a new `Block`. An empty `$id` throws `\InvalidArgumentException`.
- `BlockTree`: `__construct(Block ...$roots)` (throws `DuplicateBlockIdException` if any id repeats anywhere in the tree), `roots(): array`, `find(string $id): ?Block`, `parentOf(string $id): ?Block`, `walk(): array` (every block, depth-first, parents before children), `ids(): array`, `replace(string $id, Block $block): self` (returns a new tree; unknown id returns an equal tree; the result is re-checked for duplicate ids).
- `Document`: `__construct(string $id, string $kind, BlockTree $tree, int $formatVersion = 1)`, readonly properties, `withTree(BlockTree $tree): self`.

- [ ] **Step 1: Write the failing tests**

```php
// tests/Pest.php helpers used from here on
// block(string $id, string $type = 'acme/box', array $settings = [], Block ...$children): Block
//   => new Block($id, BlockName::fromString($type), 1, $settings, [], [], $children)

test('a block type round-trips through an array', function () {
    $data = ['name' => 'acme/field', 'version' => 2, 'title' => 'Field', 'description' => 'd', 'category' => 'forms',
        'keywords' => ['input'], 'icon' => 'text', 'settings' => ['type' => 'object', 'properties' => ['label' => ['type' => 'string']]],
        'view' => null, 'children' => false, 'parents' => ['acme/form'], 'provides' => [], 'uses' => ['acme/form-id'],
        'assets' => ['acme-field'], 'examples' => [['label' => 'Name']]];
    expect(BlockType::fromArray($data)->toArray())->toBe($data);
});
test('a block type needs a name, a title and a positive version', function (array $data) {
    expect(fn () => BlockType::fromArray($data))->toThrow(InvalidBlockTypeException::class);
})->with([[['title' => 'T']], [['name' => 'acme/x']], [['name' => 'acme/x', 'title' => 'T', 'version' => 0]]]);
test('children and parent policies', function () {
    $form = BlockName::fromString('acme/form');
    expect(ChildrenPolicy::none()->allows($form))->toBeFalse()
        ->and(ChildrenPolicy::any()->allows($form))->toBeTrue()
        ->and(ChildrenPolicy::fromArray(['acme/form'])->allows($form))->toBeTrue()
        ->and(ParentPolicy::any()->allows(null))->toBeTrue()
        ->and(ParentPolicy::only($form)->allows(null))->toBeFalse()
        ->and(ParentPolicy::only($form)->allows($form))->toBeTrue();
});
test('setting paths address settings and decorator extensions', function () {
    $own = SettingPath::fromString('style.color');
    $ext = SettingPath::fromString('@acme/visibility:rules.0');
    expect($own->namespace())->toBeNull()->and($own->segments())->toBe(['style', 'color'])
        ->and($ext->namespace())->toBe('acme/visibility')->and($ext->segments())->toBe(['rules', '0'])
        ->and($ext->toString())->toBe('@acme/visibility:rules.0');
});
test('malformed setting paths are rejected', function (string $path) {
    expect(fn () => SettingPath::fromString($path))->toThrow(InvalidSettingPathException::class);
})->with(['', '.a', 'a..b', '@:a', '@Bad/Ns:a', '@acme/x:']);
test('a tree rejects duplicate ids at any depth', function () {
    expect(fn () => new BlockTree(block('a', children: block('b')), block('b')))->toThrow(DuplicateBlockIdException::class);
});
test('a tree finds blocks, their parents, and walks parents first', function () {
    $tree = new BlockTree(block('a', 'acme/box', [], block('b'), block('c', 'acme/box', [], block('d'))), block('e'));
    expect($tree->find('d')->id)->toBe('d')->and($tree->parentOf('d')->id)->toBe('c')
        ->and($tree->parentOf('a'))->toBeNull()->and($tree->find('zz'))->toBeNull()
        ->and(array_map(fn (Block $b) => $b->id, $tree->walk()))->toBe(['a', 'b', 'c', 'd', 'e']);
});
test('replace returns a new tree and leaves the original untouched', function () {
    $tree = new BlockTree(block('a', 'acme/box', [], block('b', 'acme/box', ['n' => 1])));
    $next = $tree->replace('b', $tree->find('b')->withSettings(['n' => 2]));
    expect($next->find('b')->settings)->toBe(['n' => 2])->and($tree->find('b')->settings)->toBe(['n' => 1]);
});
```

- [ ] **Step 2: Run `vendor/bin/pest --colors=never` in `block-domain`.** Expected: FAIL, classes not found.
- [ ] **Step 3: Implement** the value objects and entities.
- [ ] **Step 4: Run `vendor/bin/pest --colors=never`.** Expected: PASS.
- [ ] **Step 5: Commit** `feat(block-domain): add block types, blocks, trees and documents`.

---

### Task 4: Registries, block ids and duplication

**Files:**
- Create in `block-domain/src/Contracts/`: `BlockTypeRegistryInterface.php`, `BlockBehaviourInterface.php`, `BlockBehaviourRegistryInterface.php`, `DecoratorRegistryInterface.php`, `SettingsMigrationInterface.php`, `SettingsMigrationRegistryInterface.php`, `DataSourceInterface.php`, `DataSourceRegistryInterface.php`, `BlockIdGeneratorInterface.php`, `BlockDuplicatorInterface.php`
- Create in `block-domain/src/ValueObjects/`: `Context.php`, `CurrentItem.php`, `DataSourceResult.php`, `AdminContext.php`, `BlockView.php`
- Create in `block-app/src/`: `Registries/{BlockTypeRegistry,BlockBehaviourRegistry,DecoratorRegistry,DataSourceRegistry,SettingsMigrationRegistry}.php`, `Ids/{RandomBlockIdGenerator,BlockDuplicator}.php`
- Test: `block-app/tests/Registries/*Test.php`, `tests/Ids/BlockDuplicatorTest.php`, `block-domain/tests/ValueObjects/ContextTest.php`

**Interfaces:**
- `Context` (immutable): `static empty(): self`, `with(string $key, mixed $value): self`, `has(string $key): bool`, `get(string $key): mixed` (null when absent), `only(array $keys): self` (keeps the current item), `withCurrentItem(?CurrentItem $item): self`, `currentItem(): ?CurrentItem`, `all(): array`.
- `CurrentItem` (readonly): `__construct(mixed $item, int $index, string $collectionId)`.
- `DataSourceResult`: `static value(mixed $value): self`, `static none(): self`, `hasValue(): bool`, `value(): mixed`.
- `AdminContext` (readonly): `__construct(array $values = [], array $issues = [], array $bindings = [], bool $hiddenOnFront = false)` (`$issues` is a list of `Issue`; `$bindings` maps a setting-path string to `['source' => string, 'used' => 'source'|'fallback'|'stored'|'default'|'none']`); `with(...)` named-argument copier.
- `BlockView` (readonly): `__construct(array $values, array $extensions = [], ?AdminContext $admin = null, ?array $childPasses = null)` (`$childPasses` null means "produce children once"; otherwise a list of `CurrentItem`, one pass each); `withExtension(string $namespace, array $values): self`, `withAdmin(?AdminContext $admin): self`.
- `BlockTypeRegistryInterface`: `add(BlockType $type): void`, `remove(BlockName $name): void`, `has(BlockName $name): bool`, `get(BlockName $name): ?BlockType`, `all(): array`.
- `BlockBehaviourInterface`: `view(BlockType $type, array $settings, Context $context, Mode $mode): BlockView`.
- `BlockBehaviourRegistryInterface`: `set(BlockName $name, BlockBehaviourInterface $behaviour): void`, `get(BlockName $name): ?BlockBehaviourInterface`.
- `DecoratorRegistryInterface`: `add(Decorator $decorator): void`, `remove(string $namespace): void`, `get(string $namespace): ?Decorator`, `for(BlockName $type): array` (list, in registration order).
- `SettingsMigrationInterface`: `from(): int`, `migrate(array $settings): array` (moves settings from version `from()` to `from() + 1`).
- `SettingsMigrationRegistryInterface`: `add(BlockName $type, SettingsMigrationInterface $migration): void`, `chain(BlockName $type, int $from, int $to): ?array` (the ordered migrations taking `$from` to `$to`; `[]` when equal; null when a step is missing or `$from > $to`).
- `DataSourceInterface`: `name(): string`, `argumentsSchema(): array`, `valueSchema(): array`, `resolve(array $arguments, Context $context, Mode $mode): DataSourceResult`.
- `DataSourceRegistryInterface`: `add(DataSourceInterface $source): void`, `remove(string $name): void`, `get(string $name): ?DataSourceInterface`.
- `BlockIdGeneratorInterface`: `generate(): string`. `RandomBlockIdGenerator` returns 16 lower-case hex characters from `random_bytes(8)`.
- `BlockDuplicatorInterface`: `duplicate(Block $block): Block` — a deep copy in which the block and every descendant has a newly generated id; everything else is unchanged.
- `BlockTypeRegistry::__construct(SchemaValidatorInterface $validator)`. `add()` calls `assertSupported($type->settingsSchema, allowPrivate: true)` and, when present, `assertSupported($type->viewSchema)`, and validates every example against the settings schema after applying defaults, throwing `InvalidBlockTypeException` naming the example index when one fails. `DecoratorRegistry::__construct(SchemaValidatorInterface $validator)` asserts the decorator's settings schema is supported (private not allowed). `DataSourceRegistry::__construct(SchemaValidatorInterface $validator)` asserts both of the source's schemas are supported.

- [ ] **Step 1: Write the failing tests**

```php
// block-app/tests/Pest.php helpers used from here on
// type(string $name, array $over = []): BlockType => BlockType::fromArray(['name' => $name, 'title' => $name] + $over)
// block(...) as in block-domain

test('registering a type with an unsupported schema keyword throws', function () {
    $registry = new BlockTypeRegistry(new SubsetSchemaValidator());
    expect(fn () => $registry->add(type('acme/x', ['settings' => ['type' => 'object', 'properties' => ['a' => ['format' => 'email']]]])))
        ->toThrow(UnsupportedSchemaException::class);
});
test('registering a type whose example fails its own schema throws', function () {
    $registry = new BlockTypeRegistry(new SubsetSchemaValidator());
    $type = type('acme/x', ['settings' => ['type' => 'object', 'required' => ['label'], 'properties' => ['label' => ['type' => 'string']]], 'examples' => [['label' => 'ok'], []]]);
    expect(fn () => $registry->add($type))->toThrow(InvalidBlockTypeException::class, '1');
});
test('types can be added, replaced, listed and removed', function () {
    $registry = new BlockTypeRegistry(new SubsetSchemaValidator());
    $registry->add(type('acme/a'));
    $registry->add(type('acme/a', ['title' => 'Second']));
    $registry->add(type('acme/b'));
    expect($registry->get(BlockName::fromString('acme/a'))->catalogue->title)->toBe('Second')
        ->and(count($registry->all()))->toBe(2);
    $registry->remove(BlockName::fromString('acme/a'));
    expect($registry->has(BlockName::fromString('acme/a')))->toBeFalse();
});
test('decorators apply to all types or to a named list', function () {
    $registry = new DecoratorRegistry(new SubsetSchemaValidator());
    $registry->add(new Decorator('acme/visibility', ['type' => 'object']));
    $registry->add(new Decorator('acme/anim', ['type' => 'object'], ['acme/box']));
    $names = fn (string $type) => array_map(fn (Decorator $d) => $d->namespace, $registry->for(BlockName::fromString($type)));
    expect($names('acme/box'))->toBe(['acme/visibility', 'acme/anim'])->and($names('acme/text'))->toBe(['acme/visibility']);
});
test('a migration chain is ordered, empty when equal, and null when a step is missing', function () {
    $registry = new SettingsMigrationRegistry();
    $name = BlockName::fromString('acme/x');
    $registry->add($name, migration(from: 2));
    $registry->add($name, migration(from: 1));
    expect(array_map(fn ($m) => $m->from(), $registry->chain($name, 1, 3)))->toBe([1, 2])
        ->and($registry->chain($name, 3, 3))->toBe([])
        ->and($registry->chain($name, 1, 4))->toBeNull()
        ->and($registry->chain($name, 3, 1))->toBeNull();
});
test('duplicating gives the block and every descendant a new id and changes nothing else', function () {
    $original = block('a', 'acme/box', ['n' => 1], block('b', 'acme/box', ['n' => 2]));
    $copy = (new BlockDuplicator(new RandomBlockIdGenerator()))->duplicate($original);
    expect($copy->id)->not->toBe('a')->and($copy->children[0]->id)->not->toBe('b')
        ->and($copy->id)->not->toBe($copy->children[0]->id)
        ->and($copy->settings)->toBe(['n' => 1])->and($copy->children[0]->settings)->toBe(['n' => 2])
        ->and(fn () => new BlockTree($original, $copy))->not->toThrow(DuplicateBlockIdException::class);
});
test('generated ids are 16 hex characters and differ', function () {
    $g = new RandomBlockIdGenerator();
    expect($g->generate())->toMatch('/^[0-9a-f]{16}\z/')->and($g->generate())->not->toBe($g->generate());
});
// block-domain/tests/ValueObjects/ContextTest.php
test('context is immutable, filters by key, and keeps the current item', function () {
    $item = new CurrentItem(['id' => 9], 0, 'q1');
    $ctx = Context::empty()->with('acme/form-id', 'f1')->with('acme/other', 2)->withCurrentItem($item);
    $only = $ctx->only(['acme/form-id']);
    expect(Context::empty()->has('acme/form-id'))->toBeFalse()
        ->and($only->all())->toBe(['acme/form-id' => 'f1'])->and($only->currentItem())->toBe($item)
        ->and($ctx->get('missing'))->toBeNull();
});
```

`migration(int $from)` is a `tests/Pest.php` helper returning an anonymous `SettingsMigrationInterface` whose `migrate()` returns its input.

- [ ] **Step 2: Run `vendor/bin/pest --colors=never` in `block-app` and `block-domain`.** Expected: FAIL, classes not found.
- [ ] **Step 3: Implement** the contracts, value objects, registries, generator and duplicator.
- [ ] **Step 4: Run `vendor/bin/pest --colors=never` in both packages.** Expected: PASS.
- [ ] **Step 5: Commit** `feat(block): add registries, block ids and duplication`.

---

### Task 5: In-memory infrastructure and its contract suites

**Files:**
- Create in `block-domain/src/Contracts/`: `DocumentRepositoryInterface.php`
- Create: `packages/block-infra-memory/{composer.json,phpunit.xml,tests/Pest.php}`, `src/{InMemoryDocumentRepository,ArrayDataSource,MemoryServiceProvider}.php` (require-dev: `speedyspec/block-app`, `speedyspec/block-contract-tests`, Pest)
- Create in `block-contract-tests/src/`: `documentRepositoryContract.php`, `dataSourceContract.php`
- Test: `block-infra-memory/tests/{ContractsTest,ArrayDataSourceTest,MemoryServiceProviderTest}.php`

**Interfaces:**
- `DocumentRepositoryInterface`: `find(string $id): ?Document`, `save(Document $document): void`, `remove(string $id): void`.
- `InMemoryDocumentRepository implements DocumentRepositoryInterface`.
- `ArrayDataSource implements DataSourceInterface`: `__construct(string $name, array $values = [], array $valueSchema = [])`, `set(string $key, mixed $value): void`. `argumentsSchema()` is `['type' => 'object', 'required' => ['key'], 'properties' => ['key' => ['type' => 'string']]]`. `resolve()` returns the value stored under `$arguments['key']`, or `none()` when the key is missing or not set. An empty `$valueSchema` means "any value" (`[]`).
- `MemoryServiceProvider::register(BlockServiceContainer $container): void` binds `DocumentRepositoryInterface`.
- `documentRepositoryContract(Closure $make)` asserts: an unknown id returns null; `save` then `find` returns a document with the same id, kind, format version and block ids; saving the same id again replaces it; `remove` then `find` returns null; removing an unknown id does not throw.
- `dataSourceContract(Closure $make, array $knownArguments, array $unknownArguments)` asserts: `name()` is a non-empty string; both schemas are accepted by `SubsetSchemaValidator::assertSupported`; `$knownArguments` validate against `argumentsSchema()` and resolve to a value that validates against `valueSchema()`; `$unknownArguments` resolve to `none()`; resolving twice with the same arguments gives the same result in both modes.
- The shared contract functions use fixed `describe()` names, so a test file that calls one more than once wraps each call in its own `describe()`.

- [ ] **Step 1: Write the failing tests**

```php
// tests/ContractsTest.php
documentRepositoryContract(fn () => new InMemoryDocumentRepository());
dataSourceContract(
    fn () => new ArrayDataSource('acme/values', ['greeting' => 'hello'], ['type' => 'string']),
    ['key' => 'greeting'],
    ['key' => 'missing'],
);
// tests/ArrayDataSourceTest.php
test('set adds a value and a missing key argument yields none', function () {
    $source = new ArrayDataSource('acme/values');
    $source->set('n', 3);
    expect($source->resolve(['key' => 'n'], Context::empty(), Mode::Front)->value())->toBe(3)
        ->and($source->resolve([], Context::empty(), Mode::Front)->hasValue())->toBeFalse();
});
test('a stored null is a value, not none', function () {
    $source = new ArrayDataSource('acme/values', ['x' => null]);
    expect($source->resolve(['key' => 'x'], Context::empty(), Mode::Admin)->hasValue())->toBeTrue();
});
// tests/MemoryServiceProviderTest.php
test('binds the document repository as a replaceable contract', function () {
    $c = new BlockServiceContainer();
    (new MemoryServiceProvider())->register($c);
    $mine = new InMemoryDocumentRepository();
    $c->add(DocumentRepositoryInterface::class, fn () => $mine);
    expect($c->get(DocumentRepositoryInterface::class))->toBe($mine);
});
```

- [ ] **Step 2: Run `composer install && vendor/bin/pest --colors=never` in `block-infra-memory`.** Expected: FAIL, classes not found.
- [ ] **Step 3: Implement** the contract, adapters, provider and the two contract functions.
- [ ] **Step 4: Run `vendor/bin/pest --colors=never`.** Expected: PASS.
- [ ] **Step 5: Commit** `feat(block-infra-memory): add in-memory repository and data source`.

---

### Task 6: Serialization and client export

**Files:**
- Create in `block-domain/src/Contracts/`: `DocumentSerializerInterface.php`, `ClientExportInterface.php`; `src/Exceptions/InvalidDocumentException.php`
- Create in `block-app/src/Serialization/`: `JsonDocumentSerializer.php`, `ClientExport.php`, `README.md` (documents the JSON shape below)
- Create in `block-contract-tests/src/`: `documentSerializerContract.php`
- Test: `block-app/tests/Serialization/{JsonDocumentSerializerTest,ClientExportTest}.php`

**Interfaces:**
- `DocumentSerializerInterface`: `serialize(Document $document): string`, `parse(string $serialized): Document` (throws `InvalidDocumentException`, which extends `\InvalidArgumentException`, for input that is not a document).
- `JsonDocumentSerializer`. JSON shape, fixed:

```json
{"format": 1, "id": "doc-1", "kind": "page", "blocks": [
  {"id": "a", "type": "acme/box", "version": 1, "settings": {}, "extensions": {}, "bindings":
    {"label": {"source": "acme/values", "arguments": {"key": "x"}, "fallback": "y"}}, "children": []}
]}
```

  `fallback` is present only when the binding has one. `settings`, `extensions`, `bindings` and `arguments` serialize as JSON objects even when empty; `children` and `blocks` as JSON arrays. Keys appear in the order shown. Encoded with `JSON_UNESCAPED_SLASHES | JSON_UNESCAPED_UNICODE | JSON_THROW_ON_ERROR`. `parse()` rejects: invalid JSON, a missing or non-1 `format`, a missing `id`/`kind`/`blocks`, a block with a missing or invalid `id`/`type`, and duplicate block ids. It needs no registry and keeps blocks of unregistered types unchanged.
- `ClientExportInterface`: `export(Document $document, Mode $mode): Document`.
- `ClientExport::__construct(BlockTypeRegistryInterface $types, SchemaValidatorInterface $validator)`. `Mode::Admin` returns the document unchanged. `Mode::Front`: for each block of a registered type, remove every private top-level setting and any binding whose path targets one; for each block of an unregistered type, set `settings` to `[]` and remove all bindings; extensions are kept; children are processed the same way.
- `documentSerializerContract(Closure $make)` asserts: `parse(serialize($doc))` equals `$doc` in id, kind, format version and, block by block, id, type, type version, settings, extensions, bindings and child order, for a document with nested children, an extension, a binding with and without a fallback, and a block of a type name no registry knows; and `parse('not a document')` throws `InvalidDocumentException`.

- [ ] **Step 1: Write the failing tests**

```php
documentSerializerContract(fn () => new JsonDocumentSerializer());

test('an unknown block type survives a load and save unchanged', function () {
    $json = '{"format":1,"id":"d","kind":"page","blocks":[{"id":"a","type":"gone/away","version":3,'
        . '"settings":{"secret":"s","n":1},"extensions":{},"bindings":{},"children":[]}]}';
    $s = new JsonDocumentSerializer();
    expect($s->serialize($s->parse($json)))->toBe($json);
});
test('parse rejects documents it cannot trust', function (string $json) {
    expect(fn () => (new JsonDocumentSerializer())->parse($json))->toThrow(InvalidDocumentException::class);
})->with([
    '{', '[]', '{"format":2,"id":"d","kind":"page","blocks":[]}', '{"format":1,"kind":"page","blocks":[]}',
    '{"format":1,"id":"d","kind":"page","blocks":[{"type":"acme/x"}]}',
    '{"format":1,"id":"d","kind":"page","blocks":[{"id":"a","type":"Bad Name"}]}',
    '{"format":1,"id":"d","kind":"page","blocks":[{"id":"a","type":"acme/x"},{"id":"a","type":"acme/x"}]}',
]);

// ClientExportTest.php — helper exporter(BlockType ...$types): ClientExport
test('front export removes private settings and bindings that target them', function () {
    $form = type('acme/form', ['settings' => ['type' => 'object', 'properties' => [
        'to' => ['type' => 'string', 'x-private' => true], 'title' => ['type' => 'string']]]]);
    $block = block('f', 'acme/form', ['to' => 'owner@example.test', 'title' => 'Contact'])
        ->withBinding(SettingPath::fromString('to'), Binding::to('acme/values', ['key' => 'x']))
        ->withBinding(SettingPath::fromString('title'), Binding::to('acme/values', ['key' => 'y']));
    $out = exporter($form)->export(new Document('d', 'page', new BlockTree($block)), Mode::Front)->tree->find('f');
    expect($out->settings)->toBe(['title' => 'Contact'])->and(array_keys($out->bindings))->toBe(['title']);
});
test('front export strips every setting of an unregistered type but keeps the block and its children', function () {
    $doc = new Document('d', 'page', new BlockTree(block('a', 'gone/away', ['secret' => 's'], block('b', 'acme/box', ['n' => 1]))));
    $out = exporter(type('acme/box'))->export($doc, Mode::Front)->tree;
    expect($out->find('a')->settings)->toBe([])->and($out->find('b')->settings)->toBe(['n' => 1]);
});
test('admin export returns everything', function () {
    $form = type('acme/form', ['settings' => ['type' => 'object', 'properties' => ['to' => ['type' => 'string', 'x-private' => true]]]]);
    $doc = new Document('d', 'page', new BlockTree(block('f', 'acme/form', ['to' => 'owner@example.test'])));
    expect(exporter($form)->export($doc, Mode::Admin)->tree->find('f')->settings)->toBe(['to' => 'owner@example.test']);
});
test('the serialized front export does not contain the private value', function () {
    $form = type('acme/form', ['settings' => ['type' => 'object', 'properties' => ['to' => ['type' => 'string', 'x-private' => true]]]]);
    $doc = new Document('d', 'page', new BlockTree(block('f', 'acme/form', ['to' => 'owner@example.test'])));
    $json = (new JsonDocumentSerializer())->serialize(exporter($form)->export($doc, Mode::Front));
    expect($json)->not->toContain('owner@example.test');
});
```

- [ ] **Step 2: Run `vendor/bin/pest --colors=never` in `block-app`.** Expected: FAIL, classes not found.
- [ ] **Step 3: Implement** the contracts, serializer, export, README and contract function.
- [ ] **Step 4: Run `vendor/bin/pest --colors=never`.** Expected: PASS.
- [ ] **Step 5: Commit** `feat(block-app): add JSON serialization and client export`.

---

### Task 7: Resolving stages 1 to 4 (prepare, gate, resolve, validate)

**Files:**
- Create in `block-domain/src/ValueObjects/`: `Issue.php`, `IssueKind.php`
- Create in `block-domain/src/Contracts/`: `PrepareStepInterface.php`, `GateStepInterface.php`, `ValidateStepInterface.php`, `StepRegistryInterface.php`
- Create in `block-app/src/`: `Registries/StepRegistry.php`, `Resolving/{PrepareStage,GateStage,ResolveStage,ValidateStage,StageOutcome}.php`
- Test: `block-app/tests/Resolving/{PrepareStageTest,GateStageTest,ResolveStageTest,ValidateStageTest}.php`

**Interfaces:**
- `enum IssueKind: string`: `UnknownType = 'unknown-type'`, `MigrationFailed = 'migration-failed'`, `UnknownDataSource = 'unknown-data-source'`, `InvalidBinding = 'invalid-binding'`, `PrivateBinding = 'private-binding'`, `InvalidSettings = 'invalid-settings'`, `DisallowedChild = 'disallowed-child'`, `InvalidView = 'invalid-view'`.
- `Issue` (readonly): `__construct(string $blockId, IssueKind $kind, string $message, ?string $path = null)`.
- `PrepareStepInterface`: `prepare(Block $block, BlockType $type, Context $context, Mode $mode): Block` — may return a block with changed settings or extensions; a returned block whose id, type or children differ is ignored in favour of the input's.
- `GateStepInterface`: `shown(Block $block, BlockType $type, Context $context, Mode $mode): bool`.
- `ValidateStepInterface`: `validate(Block $block, BlockType $type, array $settings, Context $context, Mode $mode): array` (list of `Issue`).
- `StepRegistryInterface`: `add(string $stepInterface, object $step, int $priority = 10): void` (throws `\InvalidArgumentException` when `$step` is not an instance of `$stepInterface`), `for(string $stepInterface): array` (ordered by priority, then insertion).
- `StageOutcome` (app-internal, readonly): `__construct(Block $block, array $settings, array $extensions, array $issues, bool $continue, bool $hiddenOnFront = false, array $bindingInfo = [])`.
- Stage classes, each with one method, all in `SpeedySpec\Block\App\Resolving`:
  - `PrepareStage::__construct(BlockTypeRegistryInterface, SettingsMigrationRegistryInterface, DecoratorRegistryInterface, SchemaValidatorInterface, StepRegistryInterface)`; `run(Block $block, Context $context, Mode $mode): StageOutcome`.
  - `GateStage::__construct(StepRegistryInterface)`; `run(StageOutcome $prepared, BlockType $type, Context $context, Mode $mode): StageOutcome`.
  - `ResolveStage::__construct(DataSourceRegistryInterface, DecoratorRegistryInterface, SchemaValidatorInterface)`; `run(StageOutcome $gated, BlockType $type, Context $context, Mode $mode): StageOutcome`.
  - `ValidateStage::__construct(BlockTypeRegistryInterface, SchemaValidatorInterface, StepRegistryInterface)`; `run(StageOutcome $resolved, BlockType $type, Context $context, Mode $mode): StageOutcome`.

**Rules the tests pin:**
- **Prepare.** Unknown type: issue `UnknownType`, `continue` false. Stored `typeVersion` below the type's version: run `chain()`; a null chain is issue `MigrationFailed`, `continue` false; a stored version above the type's is the same. After migrating, the outcome's block carries the type's current version. Then registered prepare steps run in order. Then defaults overlay settings (`withDefaults`), and for each decorator that applies to the type, defaults overlay that namespace's extension settings (an absent extension becomes the decorator's defaults). Extensions for namespaces with no applicable decorator are left as stored.
- **Gate.** No steps: shown. Any step returning false: in `Front`, `continue` false with no issue; in `Admin`, `continue` true and `hiddenOnFront` true.
- **Resolve.** For each binding, in stored order: parse the path; look up the subschema (the block's settings schema, or the decorator's for a namespaced path). A path with no subschema is issue `InvalidBinding`, binding skipped. A path whose first segment is a private setting is issue `PrivateBinding`, binding skipped. An unregistered source is issue `UnknownDataSource`; arguments that fail the source's `argumentsSchema()` are issue `InvalidBinding`; in both cases the source candidate is treated as absent. Candidates are tried in order: source value, binding fallback (when it has one), the current setting at that path (when set), the schema default (when declared). The first that validates against the subschema is written at the path. Each rejected candidate is issue `InvalidBinding`. `bindingInfo[path]` records `['source' => name, 'used' => 'source'|'fallback'|'stored'|'default'|'none']`. No data source is called when Gate stopped the block.
- **Validate.** Settings must validate against the settings schema, and each applicable decorator's extension against its schema; any failure is issue `InvalidSettings` (with the error's path, prefixed `@namespace:` for an extension) and `continue` false. Then registered validate steps run and their issues are appended (they do not stop the block). Children are checked by the caller in Task 8.

- [ ] **Step 1: Write the failing tests**

```php
// tests/Pest.php: helper stages(?callable $setup = null): object returning an object with public properties
// types, migrations, decorators, sources, steps (the registries) and prepare, gate, resolve, validate (the stages),
// all sharing one SubsetSchemaValidator. $setup receives that object to register things.
// helper kinds(array $issues): array => array_map(fn (Issue $i) => $i->kind, $issues)

$field = fn () => type('acme/field', ['version' => 2, 'settings' => ['type' => 'object', 'required' => ['label'], 'properties' => [
    'label' => ['type' => 'string', 'maxLength' => 10, 'default' => 'Untitled'],
    'value' => ['type' => 'string', 'maxLength' => 5, 'default' => ''],
    'to' => ['type' => 'string', 'x-private' => true],
]]]);

test('prepare stops at an unknown type', function () {
    $out = stages()->prepare->run(block('a', 'gone/away'), Context::empty(), Mode::Front);
    expect($out->continue)->toBeFalse()->and(kinds($out->issues))->toBe([IssueKind::UnknownType]);
});
test('prepare migrates old settings, then overlays defaults', function () use ($field) {
    $s = stages(function ($s) use ($field) {
        $s->types->add($field());
        $s->migrations->add(BlockName::fromString('acme/field'), new class implements SettingsMigrationInterface {
            public function from(): int { return 1; }
            public function migrate(array $settings): array { return ['label' => $settings['title'] ?? 'x']; }
        });
    });
    $out = $s->prepare->run(block('a', 'acme/field', ['title' => 'Name']), Context::empty(), Mode::Front);
    expect($out->settings)->toBe(['label' => 'Name', 'value' => ''])->and($out->block->typeVersion)->toBe(2)->and($out->continue)->toBeTrue();
});
test('prepare fails when a migration step is missing or the stored version is newer', function (int $stored) use ($field) {
    $s = stages(fn ($s) => $s->types->add($field()));
    $out = $s->prepare->run(block('a', 'acme/field')->withTypeVersion($stored), Context::empty(), Mode::Front);
    expect($out->continue)->toBeFalse()->and(kinds($out->issues))->toBe([IssueKind::MigrationFailed]);
})->with([1, 3]);
test('prepare fills decorator defaults for applicable decorators only', function () use ($field) {
    $s = stages(function ($s) use ($field) {
        $s->types->add($field());
        $s->decorators->add(new Decorator('acme/vis', ['type' => 'object', 'properties' => ['hide' => ['type' => 'boolean', 'default' => false]]]));
        $s->decorators->add(new Decorator('acme/anim', ['type' => 'object', 'properties' => ['fx' => ['type' => 'string', 'default' => 'none']]], ['acme/box']));
    });
    $out = $s->prepare->run(block('a', 'acme/field')->withTypeVersion(2), Context::empty(), Mode::Front);
    expect($out->extensions)->toBe(['acme/vis' => ['hide' => false]]);
});
test('a gate step hides a block on the front end and only marks it in the admin', function (Mode $mode, bool $continue, bool $hidden) use ($field) {
    $s = stages(function ($s) use ($field) {
        $s->types->add($field());
        $s->steps->add(GateStepInterface::class, new class implements GateStepInterface {
            public function shown(Block $block, BlockType $type, Context $context, Mode $mode): bool { return false; }
        });
    });
    $prepared = $s->prepare->run(block('a', 'acme/field')->withTypeVersion(2), Context::empty(), $mode);
    $out = $s->gate->run($prepared, $field(), Context::empty(), $mode);
    expect($out->continue)->toBe($continue)->and($out->hiddenOnFront)->toBe($hidden)->and($out->issues)->toBe([]);
})->with([[Mode::Front, false, false], [Mode::Admin, true, true]]);
test('a bound value replaces the stored setting', function () use ($field) {
    $s = stages(function ($s) use ($field) {
        $s->types->add($field());
        $s->sources->add(new ArrayDataSource('acme/values', ['v' => 'abc']));
    });
    $block = block('a', 'acme/field', ['label' => 'L'])->withTypeVersion(2)
        ->withBinding(SettingPath::fromString('value'), Binding::to('acme/values', ['key' => 'v']));
    $out = $s->resolve->run($s->prepare->run($block, Context::empty(), Mode::Front), $field(), Context::empty(), Mode::Front);
    expect($out->settings['value'])->toBe('abc')->and($out->bindingInfo['value'])->toBe(['source' => 'acme/values', 'used' => 'source']);
});
test('a rejected bound value falls back and never stops the block', function (array $stored, ?string $fallback, string $expected, string $used) use ($field) {
    $s = stages(function ($s) use ($field) {
        $s->types->add($field());
        $s->sources->add(new ArrayDataSource('acme/values', ['v' => 'far too long']));
    });
    $binding = Binding::to('acme/values', ['key' => 'v']);
    $block = block('a', 'acme/field', $stored)->withTypeVersion(2)
        ->withBinding(SettingPath::fromString('value'), null === $fallback ? $binding : $binding->withFallback($fallback));
    $out = $s->resolve->run($s->prepare->run($block, Context::empty(), Mode::Front), $field(), Context::empty(), Mode::Front);
    expect($out->settings['value'])->toBe($expected)->and($out->bindingInfo['value']['used'])->toBe($used)
        ->and($out->continue)->toBeTrue()->and(kinds($out->issues))->toContain(IssueKind::InvalidBinding);
})->with([
    [['label' => 'L'], 'fb', 'fb', 'fallback'],
    [['label' => 'L', 'value' => 'kept'], null, 'kept', 'stored'],
    [['label' => 'L'], 'also too long', '', 'stored'],
]);
test('a binding can target a decorator setting through a namespaced path', function () use ($field) {
    $s = stages(function ($s) use ($field) {
        $s->types->add($field());
        $s->decorators->add(new Decorator('acme/vis', ['type' => 'object', 'properties' => ['hide' => ['type' => 'boolean', 'default' => false]]]));
        $s->sources->add(new ArrayDataSource('acme/values', ['h' => true]));
    });
    $block = block('a', 'acme/field', ['label' => 'L'])->withTypeVersion(2)
        ->withBinding(SettingPath::fromString('@acme/vis:hide'), Binding::to('acme/values', ['key' => 'h']));
    $out = $s->resolve->run($s->prepare->run($block, Context::empty(), Mode::Front), $field(), Context::empty(), Mode::Front);
    expect($out->extensions['acme/vis']['hide'])->toBeTrue()->and($out->settings)->not->toHaveKey('hide');
});
test('a private setting cannot be bound', function () use ($field) {
    $s = stages(function ($s) use ($field) {
        $s->types->add($field());
        $s->sources->add(new ArrayDataSource('acme/values', ['v' => 'attacker@example.test']));
    });
    $block = block('a', 'acme/field', ['label' => 'L', 'to' => 'owner@example.test'])->withTypeVersion(2)
        ->withBinding(SettingPath::fromString('to'), Binding::to('acme/values', ['key' => 'v']));
    $out = $s->resolve->run($s->prepare->run($block, Context::empty(), Mode::Front), $field(), Context::empty(), Mode::Front);
    expect($out->settings['to'])->toBe('owner@example.test')->and(kinds($out->issues))->toBe([IssueKind::PrivateBinding]);
});
test('an unknown data source and an undeclared path are reported and skipped', function () use ($field) {
    $s = stages(fn ($s) => $s->types->add($field()));
    $block = block('a', 'acme/field', ['label' => 'L'])->withTypeVersion(2)
        ->withBinding(SettingPath::fromString('value'), Binding::to('nope/source'))
        ->withBinding(SettingPath::fromString('missing'), Binding::to('nope/source'));
    $out = $s->resolve->run($s->prepare->run($block, Context::empty(), Mode::Front), $field(), Context::empty(), Mode::Front);
    expect(kinds($out->issues))->toBe([IssueKind::UnknownDataSource, IssueKind::InvalidBinding])->and($out->continue)->toBeTrue();
});
test('invalid settings stop the block and name the path', function () use ($field) {
    $s = stages(fn ($s) => $s->types->add($field()));
    $block = block('a', 'acme/field', ['label' => 'this label is far too long'])->withTypeVersion(2);
    $resolved = $s->resolve->run($s->prepare->run($block, Context::empty(), Mode::Front), $field(), Context::empty(), Mode::Front);
    $out = $s->validate->run($resolved, $field(), Context::empty(), Mode::Front);
    expect($out->continue)->toBeFalse()->and($out->issues[0]->kind)->toBe(IssueKind::InvalidSettings)->and($out->issues[0]->path)->toBe('label');
});
test('step registry orders by priority then insertion and rejects the wrong type', function () {
    $r = new StepRegistry();
    $a = gateStep(true); $b = gateStep(true); $c = gateStep(true);
    $r->add(GateStepInterface::class, $a, 20);
    $r->add(GateStepInterface::class, $b, 10);
    $r->add(GateStepInterface::class, $c, 10);
    expect($r->for(GateStepInterface::class))->toBe([$b, $c, $a])
        ->and(fn () => $r->add(GateStepInterface::class, new stdClass()))->toThrow(InvalidArgumentException::class);
});
```

`gateStep(bool $shown)` is a `tests/Pest.php` helper returning an anonymous `GateStepInterface`. `block-app` adds `speedyspec/block-infra-memory` to `require-dev` for `ArrayDataSource`.

- [ ] **Step 2: Run `vendor/bin/pest --colors=never` in `block-app`.** Expected: FAIL, classes not found.
- [ ] **Step 3: Implement** the domain types, step registry and the four stages.
- [ ] **Step 4: Run `vendor/bin/pest --colors=never`.** Expected: PASS.
- [ ] **Step 5: Commit** `feat(block-app): add prepare, gate, resolve and validate stages`.

---

### Task 8: View stage, context and the document resolver

**Files:**
- Create in `block-domain/src/ValueObjects/`: `ResolvedBlock.php`, `ResolvedDocument.php`
- Create in `block-domain/src/Contracts/`: `ViewStepInterface.php`, `DocumentResolverInterface.php`
- Create in `block-app/src/`: `Resolving/{ViewStage,DocumentResolver}.php`, `AppServiceProvider.php`
- Test: `block-app/tests/Resolving/{ViewStageTest,DocumentResolverTest}.php`, `tests/AppServiceProviderTest.php`

**Interfaces:**
- `ResolvedBlock` (readonly): `__construct(Block $block, ?BlockType $type, array $settings, array $extensions, ?BlockView $view, Context $context, array $passes = [])`. `$block` is the prepared block; `$context` is the context the block received; `$passes` is a list of passes, each a list of `ResolvedBlock` children resolved with that pass's context. `isResolved(): bool` is true when `$view` is not null. `children(): array` returns the first pass, or `[]`.
- `ResolvedDocument` (readonly): `__construct(Document $document, Mode $mode, array $roots, array $issues)`; `find(string $id): ?ResolvedBlock` (first occurrence, depth-first through first passes), `issuesFor(string $id): array`, `hasIssues(): bool`.
- `ViewStepInterface`: `extend(BlockView $view, Block $block, BlockType $type, array $extensions, Context $context, Mode $mode): BlockView` — may only add or replace entries under `extensions`; a returned view whose `values`, `admin` or `childPasses` differ is ignored in favour of the input's.
- `ViewStage::__construct(BlockBehaviourRegistryInterface, SchemaValidatorInterface, StepRegistryInterface)`; `run(StageOutcome $validated, BlockType $type, Context $context, Mode $mode): array{0: ?BlockView, 1: array}` (the view or null, and issues).
- `DocumentResolverInterface`: `resolve(Document $document, Mode $mode, ?Context $context = null): ResolvedDocument`.
- `DocumentResolver::__construct(PrepareStage, GateStage, ResolveStage, ValidateStage, ViewStage, BlockTypeRegistryInterface)`.
- `AppServiceProvider::register(BlockServiceContainer $container): void` binds, one `add()` each with collaborators fetched lazily inside the factory: `SchemaValidatorInterface`, `BlockTypeRegistryInterface`, `BlockBehaviourRegistryInterface`, `DecoratorRegistryInterface`, `SettingsMigrationRegistryInterface`, `DataSourceRegistryInterface`, `StepRegistryInterface`, `BlockIdGeneratorInterface`, `BlockDuplicatorInterface`, `DocumentSerializerInterface`, `ClientExportInterface`, `DocumentResolverInterface`.

**Rules the tests pin:**
- **View.** The behaviour receives the final settings and `Context::only($type->uses)`. With no behaviour the view's values are the final settings. In `Front` mode private settings are removed from a default view's values, and `admin` is forced to null whatever the behaviour returned. In `Admin` mode the view's `AdminContext` is the behaviour's (or an empty one) with the system's additions: this block's issues, `bindingInfo`, and `hiddenOnFront`. When the type declares a view schema, the view's values must validate against it, else issue `InvalidView` and no view. Then registered view steps run.
- **Context for children.** Start from the context the block received; for each `provides` entry `key => viewKey`, when the view's values have `viewKey`, set `key`. A missing view key is skipped.
- **Children.** A child is skipped with issue `DisallowedChild` (on the child's id) when the parent type's children policy does not allow its type, or when the child's type is registered and its parent policy does not allow the parent's type. A root block whose registered type's parent policy does not allow a null parent is skipped the same way. A child of an unregistered type is passed on and gets `UnknownType` from Prepare.
- **Passes.** `childPasses` null: one pass with the child context. Otherwise one pass per `CurrentItem`, each with `withCurrentItem()`; an empty list means children are not resolved.
- **Stopping.** A block stopped at any stage is a `ResolvedBlock` with a null view and no passes; its children are not resolved and no data source is called for them. Issues from every block are collected, in tree order, on the `ResolvedDocument`.

- [ ] **Step 1: Write the failing tests**

```php
// tests/Pest.php: helper resolver(?callable $setup = null): array{0: DocumentResolverInterface, 1: BlockServiceContainer}
//   => container with MemoryServiceProvider and AppServiceProvider registered; $setup receives the container.
// helper doc(Block ...$roots): Document => new Document('d', 'page', new BlockTree(...$roots))
// helper types(BlockServiceContainer $c): BlockTypeRegistryInterface, sources($c), behaviours($c), steps($c)

$form = fn () => type('acme/form', ['children' => ['acme/field'], 'provides' => ['acme/form-id' => 'formId'],
    'settings' => ['type' => 'object', 'properties' => [
        'formId' => ['type' => 'string', 'default' => 'f1'], 'to' => ['type' => 'string', 'x-private' => true, 'default' => 'owner@example.test']]]]);
$field = fn () => type('acme/field', ['parents' => ['acme/form'], 'uses' => ['acme/form-id'],
    'settings' => ['type' => 'object', 'properties' => ['key' => ['type' => 'string'], 'required' => ['type' => 'boolean', 'default' => false]]]]);

test('resolves a form and its fields without drawing anything', function () use ($form, $field) {
    [$resolver, $c] = resolver(function ($c) use ($form, $field) { types($c)->add($form()); types($c)->add($field()); });
    $result = $resolver->resolve(doc(block('f', 'acme/form', [], block('n', 'acme/field', ['key' => 'name']))), Mode::Front);
    expect($result->hasIssues())->toBeFalse()
        ->and($result->find('n')->settings)->toBe(['key' => 'name', 'required' => false])
        ->and($result->find('n')->context->get('acme/form-id'))->toBe('f1');
});
test('a private setting is absent from a front view and present in an admin view', function () use ($form, $field) {
    [$resolver] = resolver(function ($c) use ($form, $field) { types($c)->add($form()); types($c)->add($field()); });
    $document = doc(block('f', 'acme/form'));
    expect($resolver->resolve($document, Mode::Front)->find('f')->view->values)->toBe(['formId' => 'f1'])
        ->and($resolver->resolve($document, Mode::Front)->find('f')->view->admin)->toBeNull()
        ->and($resolver->resolve($document, Mode::Admin)->find('f')->view->values['to'])->toBe('owner@example.test');
});
test('a behaviour sees only the context keys its type uses', function () use ($form, $field) {
    $seen = null;
    [$resolver] = resolver(function ($c) use ($form, $field, &$seen) {
        types($c)->add($form()); types($c)->add($field());
        behaviours($c)->set(BlockName::fromString('acme/field'), new class($seen) implements BlockBehaviourInterface {
            public function __construct(private mixed &$seen) {}
            public function view(BlockType $type, array $settings, Context $context, Mode $mode): BlockView
            {
                $this->seen = $context->all();
                return new BlockView(['key' => $settings['key']], admin: new AdminContext(['hint' => 'h']));
            }
        });
    });
    $context = Context::empty()->with('acme/unrelated', 1);
    $result = $resolver->resolve(doc(block('f', 'acme/form', [], block('n', 'acme/field', ['key' => 'name']))), Mode::Front, $context);
    expect($seen)->toBe(['acme/form-id' => 'f1'])->and($result->find('n')->view->admin)->toBeNull();
});
test('the admin view carries the block admin context plus issues, bindings and the hidden flag', function () use ($form, $field) {
    [$resolver] = resolver(function ($c) use ($form, $field) {
        types($c)->add($form()); types($c)->add($field());
        sources($c)->add(new ArrayDataSource('acme/values', ['k' => 'email']));
        steps($c)->add(GateStepInterface::class, gateStep(false));
    });
    $fieldBlock = block('n', 'acme/field')->withBinding(SettingPath::fromString('key'), Binding::to('acme/values', ['key' => 'k']));
    $admin = $resolver->resolve(doc(block('f', 'acme/form', [], $fieldBlock)), Mode::Admin)->find('n')->view->admin;
    expect($admin->hiddenOnFront)->toBeTrue()->and($admin->bindings['key'])->toBe(['source' => 'acme/values', 'used' => 'source']);
});
test('a hidden block and its children are not resolved on the front end and call no data source', function () use ($form, $field) {
    $calls = 0;
    [$resolver] = resolver(function ($c) use ($form, $field, &$calls) {
        types($c)->add($form()); types($c)->add($field());
        sources($c)->add(countingSource('acme/count', $calls));
        steps($c)->add(GateStepInterface::class, gateStep(false));
    });
    $child = block('n', 'acme/field')->withBinding(SettingPath::fromString('key'), Binding::to('acme/count'));
    $result = $resolver->resolve(doc(block('f', 'acme/form', [], $child)), Mode::Front);
    expect($result->find('f')->isResolved())->toBeFalse()->and($result->find('n'))->toBeNull()->and($calls)->toBe(0)->and($result->hasIssues())->toBeFalse();
});
test('disallowed children and misplaced roots are skipped with an issue', function () use ($form, $field) {
    [$resolver] = resolver(function ($c) use ($form, $field) { types($c)->add($form()); types($c)->add($field()); types($c)->add(type('acme/box')); });
    $result = $resolver->resolve(doc(block('f', 'acme/form', [], block('x', 'acme/box')), block('n', 'acme/field')), Mode::Front);
    expect(array_map(fn (Issue $i) => [$i->blockId, $i->kind], $result->issues))
        ->toBe([['x', IssueKind::DisallowedChild], ['n', IssueKind::DisallowedChild]])
        ->and($result->find('f')->isResolved())->toBeTrue();
});
test('one bad block does not stop its siblings', function () use ($form, $field) {
    [$resolver] = resolver(function ($c) { types($c)->add(type('acme/box')); });
    $result = $resolver->resolve(doc(block('a', 'gone/away'), block('b', 'acme/box')), Mode::Front);
    expect($result->find('a')->isResolved())->toBeFalse()->and($result->find('b')->isResolved())->toBeTrue()
        ->and($result->issuesFor('a')[0]->kind)->toBe(IssueKind::UnknownType);
});
test('a view that fails its declared schema yields no view', function () {
    [$resolver] = resolver(function ($c) {
        types($c)->add(type('acme/count', ['view' => ['type' => 'object', 'required' => ['total'], 'properties' => ['total' => ['type' => 'integer']]]]));
    });
    $result = $resolver->resolve(doc(block('a', 'acme/count')), Mode::Front);
    expect($result->find('a')->isResolved())->toBeFalse()->and($result->issues[0]->kind)->toBe(IssueKind::InvalidView);
});
test('a view can ask for its children once per item', function () {
    [$resolver] = resolver(function ($c) {
        types($c)->add(type('acme/loop', ['children' => true]));
        types($c)->add(type('acme/item'));
        behaviours($c)->set(BlockName::fromString('acme/loop'), new class implements BlockBehaviourInterface {
            public function view(BlockType $type, array $settings, Context $context, Mode $mode): BlockView
            {
                return new BlockView([], childPasses: [new CurrentItem('x', 0, 'q'), new CurrentItem('y', 1, 'q')]);
            }
        });
    });
    $loop = $resolver->resolve(doc(block('l', 'acme/loop', [], block('i', 'acme/item'))), Mode::Front)->find('l');
    expect(count($loop->passes))->toBe(2)
        ->and($loop->passes[0][0]->context->currentItem()->item)->toBe('x')
        ->and($loop->passes[1][0]->context->currentItem()->index)->toBe(1);
});
test('a view step can add extension values but not change the block view values', function () {
    [$resolver] = resolver(function ($c) {
        types($c)->add(type('acme/box'));
        steps($c)->add(ViewStepInterface::class, new class implements ViewStepInterface {
            public function extend(BlockView $view, Block $block, BlockType $type, array $extensions, Context $context, Mode $mode): BlockView
            {
                return new BlockView(['hijacked' => true], ['acme/vis' => ['class' => 'is-hidden']]);
            }
        });
    });
    $view = $resolver->resolve(doc(block('a', 'acme/box', ['n' => 1])), Mode::Front)->find('a')->view;
    expect($view->values)->toBe(['n' => 1])->and($view->extensions)->toBe(['acme/vis' => ['class' => 'is-hidden']]);
});
test('every application contract resolves and can be replaced alone', function () {
    $c = new BlockServiceContainer();
    (new MemoryServiceProvider())->register($c);
    (new AppServiceProvider())->register($c);
    $mine = new RandomBlockIdGenerator();
    $c->add(BlockIdGeneratorInterface::class, fn () => $mine);
    foreach ([SchemaValidatorInterface::class, BlockTypeRegistryInterface::class, BlockBehaviourRegistryInterface::class,
        DecoratorRegistryInterface::class, SettingsMigrationRegistryInterface::class, DataSourceRegistryInterface::class,
        StepRegistryInterface::class, BlockDuplicatorInterface::class, DocumentSerializerInterface::class,
        ClientExportInterface::class, DocumentResolverInterface::class] as $contract) {
        expect($c->get($contract))->toBeInstanceOf($contract);
    }
    expect($c->get(BlockIdGeneratorInterface::class))->toBe($mine);
});
```

`countingSource(string $name, int &$calls)` is a `tests/Pest.php` helper returning an anonymous `DataSourceInterface` that increments `$calls` in `resolve()` and returns `DataSourceResult::value('x')`.

- [ ] **Step 2: Run `vendor/bin/pest --colors=never` in `block-app`.** Expected: FAIL, classes not found.
- [ ] **Step 3: Implement** the value objects, contracts, view stage, resolver and provider.
- [ ] **Step 4: Run `vendor/bin/pest --colors=never` in `block-domain`, `block-app` and `block-infra-memory`.** Expected: PASS in each.
- [ ] **Step 5: Commit** `feat(block-app): add view stage and document resolver`.

---

## Out of scope for this plan

Stages 6 and 7, output renderers, escaping, the attribute bag, the block package entry point and its shared suite (part 1b); catalogue, placement, editing operations, preview and block templates (part 1c); the HTTP API, presentation vocabulary, conditions, loops beyond the child-pass mechanism, structural blocks, document references, the WordPress adapter, and forms.
