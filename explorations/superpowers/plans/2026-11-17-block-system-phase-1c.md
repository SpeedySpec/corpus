# Block System Part 1c (builder contracts) Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Give any builder UI everything it needs to list, place, configure, preview and arrange blocks it has never seen, through contracts that work on any block and never let a builder save a broken document.

**Architecture:** Builds on parts 1a and 1b in `packages/speedyspec-api-block/`. `block-domain` gains the builder contracts: positions and placement rules, a block catalogue, block templates, validated editing operations, and preview. `block-app` gains their framework-free defaults and binds each separately. `block-infra-memory` gains a template repository. No UI and no transport are built: these are PHP contracts a builder, or a later HTTP layer, calls.

**Tech Stack:** PHP 8.4, Composer path repositories, Pest 3/4.

**Spec:** `docs/superpowers/specs/2026-11-17-block-model-design.md` (in the `wordpress-develop` checkout). Sections "What a builder uses", "Block template", "The three roles", and the "For part 1c" list under "Deferred from part 1b" are what this plan implements.

## Global Constraints

- Work only inside `/Users/jacobsantos/repos/respite-studios/wordpress-develop/packages/speedyspec-api-block/`, a clone of `SpeedySpec/speedyspec-api-block`. Never run git against the surrounding `wordpress-develop` repository and never change files outside this directory. Never push.
- Branch `feat/block-phase-1c`. Cut it from `origin/main` if part 1b's pull request (#2) has been merged; otherwise cut it from `feat/block-phase-1b`. One commit per task. Never commit `vendor/`, `composer.lock` or `.phpunit.cache/`.
- PHP `>=8.4`; every PHP file starts with `declare(strict_types=1);`; 4-space indentation.
- Braces: the opening brace of every class, interface, enum, function, method and control structure (`if`, `else`, `foreach`, `for`, `while`, `switch`, `try`, `catch`) goes on its own line, including multi-line signatures and empty bodies. Closures, arrow functions, `match` and anonymous classes keep their brace on the same line.
- Docblocks follow `/Users/jacobsantos/repos/respite-studios/wordpress-develop/.claude/rules/coding-style.md`: a summary line that does not start with "Returns", the "why" not the "how", no `@return void`, `@since 0.1.0`, no tag that only restates a non-array type, lines at most 120 characters.
- `block-domain` requires only PHP and has no `final` classes. No package references a WordPress function, class or package. `block-app`, `block-infra-memory` and `block-example` require only `speedyspec/block-domain` at runtime. No `src` file names a class from another implementation package; services reach each other only through `BlockServiceContainer`, one `add()` per contract with collaborators fetched lazily.
- Tests are Pest, files end `Test.php`, run with `vendor/bin/pest --colors=never` inside the package. Output must be free of warnings and deprecations. After adding a file to an `autoload.files` list, run `composer update <that package>` in each dependent package.
- An editing operation never throws for a bad request. It returns a refusal with reasons. It never returns a document that breaks a placement rule or whose changed settings fail their schema.
- A builder is never trusted with ids: every block an editing operation adds gets ids from the system.
- Nothing in this part checks permissions; the host decides who may call these contracts and who may ask for admin mode.

## Review Focus

Each line has a pinning test in the task named.

1. **A builder sending a broken edit** (a block where its type is not allowed, settings that fail the schema, a position that does not exist, a block moved inside itself). Expected: a refusal naming the reason, and the document unchanged. Task 4.
2. **A builder supplying its own block ids** (pasting a subtree whose ids collide with the document, or inserting a template twice). Expected: every added block gets a new id; no duplicate ids; the original subtree and template are untouched. Tasks 3 and 4.
3. **A binding set through the builder to a private setting, an unknown data source, or with arguments the source rejects.** Expected: refused, so visitor input can never be routed to a private setting by an editing mistake. Task 4.
4. **Previewing a block that is broken, hidden or missing.** Expected: a broken or hidden block still previews in admin mode with its issues and a placeholder; a missing id gives no preview; front-mode preview shows exactly what a visitor would get. Task 5.
5. **The resolver and the builder disagreeing about placement.** Expected: one placement contract decides both, so a document the editor accepts never loses a block to a placement issue when resolved. Tasks 1 and 4.

---

## File Structure

```
packages/
├── block-domain/src/
│   ├── Contracts/     PlacementRulesInterface, BlockCatalogueInterface, BlockTemplateRegistryInterface,
│   │                  BlockTemplateRepositoryInterface, DocumentEditorInterface, BlockPreviewInterface,
│   │                  UnresolvedBlockOutputInterface
│   │                  (BlockPackageRegistrarInterface gains template(); BlockPackageLoaderInterface gains packageOf())
│   ├── ValueObjects/  Position, PlacementDecision, PlacementRefusal, CatalogueEntry, BlockTemplate, EditResult,
│   │                  EditRefusal, EditRefusalKind, BlockPreview
│   ├── Entities/      BlockTree gains insert() and remove()
│   └── Exceptions/    InvalidPositionException, InvalidBlockTemplateException
├── block-app/src/
│   ├── Building/      PlacementRules, BlockCatalogue, BlockTemplateRegistry, DocumentEditor, BlockPreviewer,
│   │                  PlaceholderOutput
│   ├── Resolving/DocumentResolver.php   (uses PlacementRulesInterface)
│   ├── Output/DocumentRenderer.php      (draws a placeholder for unresolved blocks in admin mode)
│   ├── Packages/      registrar and loader gain templates and package ownership
│   └── AppServiceProvider.php           (new bindings)
├── block-infra-memory/src/   InMemoryBlockTemplateRepository; MemoryServiceProvider binds it
└── block-contract-tests/src/ blockTemplateRepositoryContract.php
```

---

### Task 1: Positions, tree editing primitives and placement rules

**Files:**
- Create in `block-domain/src/`: `ValueObjects/{Position,PlacementDecision,PlacementRefusal}.php`, `Contracts/PlacementRulesInterface.php`, `Exceptions/InvalidPositionException.php`
- Modify: `block-domain/src/Entities/BlockTree.php` (add `insert`, `remove`)
- Create: `block-app/src/Building/PlacementRules.php`
- Modify: `block-app/src/Resolving/DocumentResolver.php`, `block-app/src/AppServiceProvider.php`
- Test: `block-domain/tests/Entities/BlockTreeEditingTest.php`, `block-domain/tests/ValueObjects/PositionTest.php`, `block-app/tests/Building/PlacementRulesTest.php`

**Interfaces:**
- `Position` (readonly): `static root(int $index): self`, `static inside(string $parentId, int $index): self`, public `?string $parentId`, `int $index`. A negative index or an empty parent id throws `InvalidPositionException` (extends `\InvalidArgumentException`).
- `BlockTree::insert(Block $block, Position $position): self` — places the block at the position among the root blocks or the named parent's children. Throws `InvalidPositionException` when the parent id is not in the tree or the index is greater than the number of siblings; `DuplicateBlockIdException` (existing) when an id of the inserted subtree is already in the tree.
- `BlockTree::remove(string $id): self` — removes the block and its descendants; an unknown id returns an equal tree.
- `BlockTree::positionOf(string $id): ?Position` — where a block sits now; null for an unknown id.
- `enum PlacementRefusal: string`: `ParentRefusesChild = 'parent-refuses-child'`, `ChildRefusesParent = 'child-refuses-parent'`, `ChildRefusesRoot = 'child-refuses-root'`.
- `PlacementDecision` (readonly): `static allowed(): self`, `static refused(PlacementRefusal $reason, string $message): self`, public `bool $allowed`, `?PlacementRefusal $reason`, `?string $message`.
- `PlacementRulesInterface`:
  - `decide(BlockName $child, ?BlockName $parent): PlacementDecision` — `$parent` null means the document root.
  - `allowedChildren(?BlockName $parent): array` — the registered block types that may be placed directly inside the parent (or at the root), as a list of `BlockName` in registration order.
- `PlacementRules::__construct(BlockTypeRegistryInterface $types)`. Rules, in this order:
  1. Parent given and registered: its children policy must allow the child's name, else `ParentRefusesChild`.
  2. Child registered: its parent policy must allow the parent's name (null for the root), else `ChildRefusesParent`, or `ChildRefusesRoot` when the parent is null.
  3. An unregistered parent or an unregistered child is not a placement problem: a rule that needs a missing type is skipped (the resolver reports the unknown type on its own).
- `DocumentResolver` takes `PlacementRulesInterface` as a new last-but-one constructor parameter (before `$budget`) and uses `decide()` wherever it checked placement itself. Its issues, messages and behaviour are unchanged: a refused block is skipped with `DisallowedChild`.

- [ ] **Step 1: Write the failing tests**

```php
// block-domain/tests/Entities/BlockTreeEditingTest.php
test('insert places a block among roots or children at an index', function () {
    $tree = new BlockTree(block('a', 'acme/box', [], block('b'), block('c')), block('d'));
    $ids = fn (BlockTree $t) => array_map(fn (Block $b) => $b->id, $t->walk());
    expect($ids($tree->insert(block('x'), Position::root(0))))->toBe(['x', 'a', 'b', 'c', 'd'])
        ->and($ids($tree->insert(block('x'), Position::root(2))))->toBe(['a', 'b', 'c', 'd', 'x'])
        ->and($ids($tree->insert(block('x'), Position::inside('a', 1))))->toBe(['a', 'b', 'x', 'c', 'd'])
        ->and($ids($tree))->toBe(['a', 'b', 'c', 'd']);
});
test('insert refuses a missing parent, an index past the end, and a colliding id', function () {
    $tree = new BlockTree(block('a', 'acme/box', [], block('b')));
    expect(fn () => $tree->insert(block('x'), Position::inside('nope', 0)))->toThrow(InvalidPositionException::class)
        ->and(fn () => $tree->insert(block('x'), Position::root(2)))->toThrow(InvalidPositionException::class)
        ->and(fn () => $tree->insert(block('x'), Position::inside('a', 2)))->toThrow(InvalidPositionException::class)
        ->and(fn () => $tree->insert(block('b'), Position::root(0)))->toThrow(DuplicateBlockIdException::class)
        ->and(fn () => $tree->insert(block('x', 'acme/box', [], block('a')), Position::root(0)))->toThrow(DuplicateBlockIdException::class);
});
test('remove takes a block and its descendants and ignores an unknown id', function () {
    $tree = new BlockTree(block('a', 'acme/box', [], block('b', 'acme/box', [], block('c'))), block('d'));
    expect($tree->remove('b')->ids())->toBe(['a', 'd'])->and($tree->remove('a')->ids())->toBe(['d'])
        ->and($tree->remove('zz')->ids())->toBe(['a', 'b', 'c', 'd'])->and($tree->ids())->toBe(['a', 'b', 'c', 'd']);
});
test('positionOf reports where a block sits', function () {
    $tree = new BlockTree(block('a', 'acme/box', [], block('b'), block('c')), block('d'));
    expect($tree->positionOf('d'))->toEqual(Position::root(1))->and($tree->positionOf('c'))->toEqual(Position::inside('a', 1))
        ->and($tree->positionOf('zz'))->toBeNull();
});
// block-domain/tests/ValueObjects/PositionTest.php
test('a position rejects a negative index and an empty parent id', function () {
    expect(fn () => Position::root(-1))->toThrow(InvalidPositionException::class)
        ->and(fn () => Position::inside('', 0))->toThrow(InvalidPositionException::class);
});

// block-app/tests/Building/PlacementRulesTest.php — helper rules(BlockType ...$types): PlacementRules
test('a parent decides which children it takes and a child which parents it accepts', function () {
    $rules = rules(type('acme/form', ['children' => ['acme/field']]), type('acme/field', ['parents' => ['acme/form']]), type('acme/box', ['children' => true]), type('acme/text'));
    $n = fn (string $name) => BlockName::fromString($name);
    expect($rules->decide($n('acme/field'), $n('acme/form'))->allowed)->toBeTrue()
        ->and($rules->decide($n('acme/text'), $n('acme/form'))->reason)->toBe(PlacementRefusal::ParentRefusesChild)
        ->and($rules->decide($n('acme/field'), $n('acme/box'))->reason)->toBe(PlacementRefusal::ChildRefusesParent)
        ->and($rules->decide($n('acme/field'), null)->reason)->toBe(PlacementRefusal::ChildRefusesRoot)
        ->and($rules->decide($n('acme/text'), null)->allowed)->toBeTrue()
        ->and($rules->decide($n('acme/text'), $n('acme/text'))->reason)->toBe(PlacementRefusal::ParentRefusesChild);
});
test('an unregistered type is not a placement problem', function () {
    $rules = rules(type('acme/box', ['children' => true]), type('acme/form', ['children' => ['acme/field']]));
    $n = fn (string $name) => BlockName::fromString($name);
    expect($rules->decide($n('gone/away'), $n('acme/box'))->allowed)->toBeTrue()
        ->and($rules->decide($n('gone/away'), null)->allowed)->toBeTrue()
        ->and($rules->decide($n('acme/box'), $n('gone/away'))->allowed)->toBeTrue()
        ->and($rules->decide($n('gone/away'), $n('acme/form'))->reason)->toBe(PlacementRefusal::ParentRefusesChild);
});
test('allowedChildren lists what may go inside a parent or at the root, in registration order', function () {
    $rules = rules(type('acme/form', ['children' => ['acme/field']]), type('acme/field', ['parents' => ['acme/form']]), type('acme/box', ['children' => true]), type('acme/text'));
    $names = fn (array $list) => array_map(fn (BlockName $n) => $n->toString(), $list);
    expect($names($rules->allowedChildren(BlockName::fromString('acme/form'))))->toBe(['acme/field'])
        ->and($names($rules->allowedChildren(BlockName::fromString('acme/box'))))->toBe(['acme/form', 'acme/box', 'acme/text'])
        ->and($names($rules->allowedChildren(null)))->toBe(['acme/form', 'acme/box', 'acme/text'])
        ->and($names($rules->allowedChildren(BlockName::fromString('acme/text'))))->toBe([]);
});
test('the resolver takes its placement decisions from the placement contract', function () {
    [$resolver] = resolver(function ($c) {
        types($c)->add(type('acme/box', ['children' => true]));
        $c->add(PlacementRulesInterface::class, fn () => new class implements PlacementRulesInterface {
            public function decide(BlockName $child, ?BlockName $parent): PlacementDecision
            {
                return $parent === null ? PlacementDecision::allowed() : PlacementDecision::refused(PlacementRefusal::ParentRefusesChild, 'no children today');
            }
            public function allowedChildren(?BlockName $parent): array { return []; }
        });
    });
    $result = $resolver->resolve(doc(block('a', 'acme/box', [], block('b', 'acme/box'))), Mode::Front);
    expect($result->find('b'))->toBeNull()->and(idsAndKinds($result->issues))->toBe([['b', IssueKind::DisallowedChild]]);
});
```

- [ ] **Step 2: Create the branch,** then run `vendor/bin/pest --colors=never` in `block-domain` and `block-app`. Expected: FAIL, classes and methods not found.
- [ ] **Step 3: Implement** the value objects, tree methods, contract, rules, resolver change and the `PlacementRulesInterface` binding.
- [ ] **Step 4: Run `vendor/bin/pest --colors=never` in `block-domain`, `block-app`, `block-infra-memory` and `block-example`.** Expected: PASS in each (every existing resolver test still passes unchanged).
- [ ] **Step 5: Commit** `feat(block): add positions, tree editing and the placement contract`.

---

### Task 2: The block catalogue and package ownership

**Files:**
- Create in `block-domain/src/`: `ValueObjects/CatalogueEntry.php`, `Contracts/BlockCatalogueInterface.php`
- Modify: `block-domain/src/Contracts/BlockPackageLoaderInterface.php` (add `packageOf`), `block-app/src/Packages/BlockPackageLoader.php`, `block-app/src/AppServiceProvider.php`
- Create: `block-app/src/Building/BlockCatalogue.php`
- Test: `block-app/tests/Building/BlockCatalogueTest.php`, addition to `block-app/tests/Packages/BlockPackageLoaderTest.php`

**Interfaces:**
- `BlockPackageLoaderInterface::packageOf(BlockName $type): ?string` — the name of the loaded package that added the type; null for a type added directly to the registry or not registered.
- `CatalogueEntry` (readonly): `__construct(BlockType $type, ?string $package)`; `toArray(): array` with keys, in this order: `name`, `title`, `description`, `category`, `keywords`, `icon`, `package`, `acceptsChildren` (true unless the children policy is none), `settings` (the settings schema, including its builder hints), `examples`.
- `BlockCatalogueInterface`:
  - `all(): array` — a `CatalogueEntry` for every registered block type, in registration order.
  - `find(BlockName $name): ?CatalogueEntry`.
  - `placeableAt(Document $document, Position $position): array` — the entries whose type may be placed at the position (inside the parent block's type, or at the root). An unknown parent id, or a parent whose type is not registered, gives an empty list.
- `BlockCatalogue::__construct(BlockTypeRegistryInterface $types, PlacementRulesInterface $placement, BlockPackageLoaderInterface $packages)`.

- [ ] **Step 1: Write the failing tests**

```php
// helper catalogue(?callable $setup = null): array{0: BlockCatalogueInterface, 1: BlockServiceContainer} (via renderer()'s container)
test('lists every block type with what a builder shows, in registration order', function () {
    [$catalogue, $c] = catalogue(function ($c) {
        $c->get(BlockPackageLoaderInterface::class)->load(package('acme/basic', function ($r) {
            $r->type(type('acme/note', ['title' => 'Note', 'description' => 'A note.', 'category' => 'text', 'keywords' => ['memo'],
                'icon' => 'note', 'children' => true, 'examples' => [[]],
                'settings' => ['type' => 'object', 'properties' => ['text' => ['type' => 'string', 'title' => 'Text', 'x-control' => 'textarea']]]]));
        }));
        types($c)->add(type('host/plain'));
    });
    expect(array_map(fn (CatalogueEntry $e) => $e->type->name->toString(), $catalogue->all()))->toBe(['acme/note', 'host/plain'])
        ->and($catalogue->find(BlockName::fromString('acme/note'))->toArray())->toBe([
            'name' => 'acme/note', 'title' => 'Note', 'description' => 'A note.', 'category' => 'text', 'keywords' => ['memo'],
            'icon' => 'note', 'package' => 'acme/basic', 'acceptsChildren' => true,
            'settings' => ['type' => 'object', 'properties' => ['text' => ['type' => 'string', 'title' => 'Text', 'x-control' => 'textarea']]],
            'examples' => [[]],
        ])
        ->and($catalogue->find(BlockName::fromString('host/plain'))->package)->toBeNull()
        ->and($catalogue->find(BlockName::fromString('gone/away')))->toBeNull();
});
test('narrows the list to what may be placed at a position', function () {
    [$catalogue] = catalogue(function ($c) {
        types($c)->add(type('acme/form', ['children' => ['acme/field']]));
        types($c)->add(type('acme/field', ['parents' => ['acme/form']]));
        types($c)->add(type('acme/text'));
    });
    $document = doc(block('f', 'acme/form'), block('t', 'acme/text'), block('u', 'gone/away'));
    $names = fn (array $entries) => array_map(fn (CatalogueEntry $e) => $e->type->name->toString(), $entries);
    expect($names($catalogue->placeableAt($document, Position::inside('f', 0))))->toBe(['acme/field'])
        ->and($names($catalogue->placeableAt($document, Position::root(0))))->toBe(['acme/form', 'acme/text'])
        ->and($names($catalogue->placeableAt($document, Position::inside('t', 0))))->toBe([])
        ->and($names($catalogue->placeableAt($document, Position::inside('u', 0))))->toBe([])
        ->and($names($catalogue->placeableAt($document, Position::inside('nope', 0))))->toBe([]);
});
// BlockPackageLoaderTest addition
test('the loader knows which package added a block type', function () {
    [, $c] = renderer();
    $loader = $c->get(BlockPackageLoaderInterface::class);
    $loader->load(package('acme/basic', fn ($r) => $r->type(type('acme/note'))));
    types($c)->add(type('host/plain'));
    expect($loader->packageOf(BlockName::fromString('acme/note')))->toBe('acme/basic')
        ->and($loader->packageOf(BlockName::fromString('host/plain')))->toBeNull()
        ->and($loader->packageOf(BlockName::fromString('gone/away')))->toBeNull();
});
```

Move the `package()` helper from `BlockPackageLoaderTest.php` into `tests/Pest.php` (guarded) so both test files share it.

- [ ] **Step 2: Run `vendor/bin/pest --colors=never` in `block-app`.** Expected: FAIL.
- [ ] **Step 3: Implement** the entry, contract, catalogue, `packageOf` and the binding.
- [ ] **Step 4: Run `vendor/bin/pest --colors=never` in `block-domain` and `block-app`.** Expected: PASS.
- [ ] **Step 5: Commit** `feat(block): add the block catalogue and package ownership`.

---

### Task 3: Block templates

**Files:**
- Create in `block-domain/src/`: `ValueObjects/BlockTemplate.php`, `Contracts/{BlockTemplateRegistryInterface,BlockTemplateRepositoryInterface}.php`, `Exceptions/InvalidBlockTemplateException.php`
- Modify: `block-domain/src/Contracts/BlockPackageRegistrarInterface.php` (add `template`), `block-app/src/Packages/{BlockPackageRegistrar,BlockPackageLoader}.php`, `block-app/src/AppServiceProvider.php`
- Create: `block-app/src/Building/BlockTemplateRegistry.php`, `block-infra-memory/src/InMemoryBlockTemplateRepository.php` (bound in `MemoryServiceProvider`), `block-contract-tests/src/blockTemplateRepositoryContract.php` (add to `autoload.files`)
- Test: `block-domain/tests/ValueObjects/BlockTemplateTest.php`, `block-app/tests/Building/BlockTemplateRegistryTest.php`, `block-infra-memory/tests/ContractsTest.php` (add the new contract), addition to `block-app/tests/Packages/BlockPackageLoaderTest.php`

**Interfaces:**
- `BlockTemplate` (readonly): `__construct(string $name, string $title, BlockTree $tree, string $description = '', string $category = '')`. `$name` follows the block-name rule (lower-case `vendor/name`). Throws `InvalidBlockTemplateException` (extends `\InvalidArgumentException`) for an invalid name, an empty title, or a tree with no root blocks.
- Two stores, because they have different owners:
  - `BlockTemplateRegistryInterface` — templates defined in code by packages or the host: `add(BlockTemplate $template): void` (replaces a same-named one), `remove(string $name): void`, `find(string $name): ?BlockTemplate`, `all(): array` (registration order). Default `BlockTemplateRegistry` in `block-app`.
  - `BlockTemplateRepositoryInterface` — templates people save while building: `save(BlockTemplate $template): void`, `remove(string $name): void`, `find(string $name): ?BlockTemplate`, `all(): array` (save order; re-saving keeps the original position). Default `InMemoryBlockTemplateRepository` in `block-infra-memory`.
- `BlockPackageRegistrarInterface::template(BlockTemplate $template): void` — the template's vendor must match the package's (`BlockPackageException` otherwise), checked while registering; it is written to the template registry only when the package finishes, like every other registration, and removed again if the load is rolled back.
- `blockTemplateRepositoryContract(Closure $make)` asserts: an unknown name returns null; `save` then `find` returns a template with the same name, title, description, category and block ids; saving the same name again replaces it and keeps its place in `all()`; `remove` then `find` is null; removing an unknown name does not throw; `all()` is in save order.

- [ ] **Step 1: Write the failing tests**

```php
// block-domain/tests/ValueObjects/BlockTemplateTest.php
test('a template needs a valid name, a title and at least one block', function (string $name, string $title, BlockTree $tree) {
    expect(fn () => new BlockTemplate($name, $title, $tree))->toThrow(InvalidBlockTemplateException::class);
})->with([
    ['Not A Name', 'T', new BlockTree(block('a'))], ['acme/hero', '', new BlockTree(block('a'))], ['acme/hero', 'Hero', new BlockTree()],
]);
test('a template keeps its tree and details', function () {
    $t = new BlockTemplate('acme/hero', 'Hero', new BlockTree(block('a'), block('b')), 'A hero section.', 'sections');
    expect($t->name)->toBe('acme/hero')->and($t->tree->ids())->toBe(['a', 'b'])->and($t->category)->toBe('sections');
});
// block-app/tests/Building/BlockTemplateRegistryTest.php
test('templates are listed in registration order and replaced by name', function () {
    $r = new BlockTemplateRegistry();
    $r->add(new BlockTemplate('acme/one', 'One', new BlockTree(block('a'))));
    $r->add(new BlockTemplate('acme/two', 'Two', new BlockTree(block('a'))));
    $r->add(new BlockTemplate('acme/one', 'First', new BlockTree(block('a'))));
    expect(array_map(fn (BlockTemplate $t) => $t->title, $r->all()))->toBe(['First', 'Two'])->and($r->find('acme/nope'))->toBeNull();
    $r->remove('acme/one');
    expect($r->find('acme/one'))->toBeNull();
});
// BlockPackageLoaderTest additions
test('a package can add templates under its own vendor', function () {
    [, $c] = renderer();
    $c->get(BlockPackageLoaderInterface::class)->load(package('acme/basic', function ($r) {
        $r->type(type('acme/note'));
        $r->template(new BlockTemplate('acme/two-notes', 'Two notes', new BlockTree(block('a', 'acme/note'), block('b', 'acme/note'))));
    }));
    expect($c->get(BlockTemplateRegistryInterface::class)->find('acme/two-notes')->title)->toBe('Two notes');
});
test('a template under another vendor is refused, and a failed load leaves no template', function () {
    [, $c] = renderer();
    $loader = $c->get(BlockPackageLoaderInterface::class);
    $tree = new BlockTree(block('a', 'acme/note'));
    expect(fn () => $loader->load(package('acme/basic', fn ($r) => $r->template(new BlockTemplate('other/x', 'X', $tree)))))->toThrow(BlockPackageException::class)
        ->and(fn () => $loader->load(package('acme/later', function ($r) use ($tree) {
            $r->template(new BlockTemplate('acme/kept', 'Kept', $tree));
            throw new RuntimeException('half way');
        })))->toThrow(RuntimeException::class)
        ->and($c->get(BlockTemplateRegistryInterface::class)->all())->toBe([]);
});
// block-infra-memory/tests/ContractsTest.php addition
describe('InMemoryBlockTemplateRepository', function () {
    blockTemplateRepositoryContract(fn () => new InMemoryBlockTemplateRepository());
});
```

- [ ] **Step 2: Run `composer update speedyspec/block-contract-tests`** in `block-app`, `block-infra-memory` and `block-example`, then `vendor/bin/pest --colors=never` in `block-domain`, `block-app` and `block-infra-memory`. Expected: FAIL.
- [ ] **Step 3: Implement** the value object, contracts, registry, repository, contract suite and the registrar, loader and provider changes.
- [ ] **Step 4: Run `vendor/bin/pest --colors=never` in all four packages.** Expected: PASS.
- [ ] **Step 5: Commit** `feat(block): add block templates`.

---

### Task 4: Validated editing operations

**Files:**
- Create in `block-domain/src/`: `ValueObjects/{EditResult,EditRefusal,EditRefusalKind}.php`, `Contracts/DocumentEditorInterface.php`
- Create: `block-app/src/Building/DocumentEditor.php`
- Modify: `block-app/src/AppServiceProvider.php`
- Test: `block-app/tests/Building/DocumentEditorTest.php`

**Interfaces:**
- `enum EditRefusalKind: string`: `UnknownBlock = 'unknown-block'`, `UnknownType = 'unknown-type'`, `UnknownDecorator = 'unknown-decorator'`, `UnknownDataSource = 'unknown-data-source'`, `InvalidPosition = 'invalid-position'`, `DisallowedPlacement = 'disallowed-placement'`, `MoveIntoSelf = 'move-into-self'`, `InvalidSettings = 'invalid-settings'`, `InvalidBinding = 'invalid-binding'`, `PrivateBinding = 'private-binding'`.
- `EditRefusal` (readonly): `__construct(EditRefusalKind $kind, string $message, ?string $blockId = null, ?string $path = null)`.
- `EditResult` (readonly): `static accepted(Document $document, array $affectedIds = []): self`, `static refused(EditRefusal ...$refusals): self` (at least one), `isAccepted(): bool`, public `?Document $document` (null when refused), `array $refusals` (list of `EditRefusal`), `array $affectedIds` (ids of blocks the operation added or changed, in tree order).
- `DocumentEditorInterface` — every method takes the current `Document` and returns an `EditResult`:
  - `insert(Document $document, BlockName $type, Position $position, array $settings = []): EditResult`
  - `insertBlocks(Document $document, array $blocks, Position $position): EditResult` (`$blocks` is a list of `Block` subtrees, placed in order starting at the position)
  - `insertTemplate(Document $document, BlockTemplate $template, Position $position): EditResult`
  - `move(Document $document, string $blockId, Position $position): EditResult`
  - `remove(Document $document, string $blockId): EditResult`
  - `duplicate(Document $document, string $blockId): EditResult`
  - `updateSettings(Document $document, string $blockId, array $settings): EditResult`
  - `updateExtension(Document $document, string $blockId, string $namespace, array $settings): EditResult`
  - `setBinding(Document $document, string $blockId, SettingPath $path, Binding $binding): EditResult`
  - `clearBinding(Document $document, string $blockId, SettingPath $path): EditResult`
- `DocumentEditor::__construct(BlockTypeRegistryInterface $types, PlacementRulesInterface $placement, SchemaValidatorInterface $validator, DecoratorRegistryInterface $decorators, DataSourceRegistryInterface $sources, BlockIdGeneratorInterface $ids, BlockDuplicatorInterface $duplicator)`.

**Rules the tests pin:**
- **Common.** An unknown block id is `UnknownBlock`. A position whose parent is not in the document, or whose index is past the end, is `InvalidPosition`. Every operation leaves the input document untouched and, when refused, returns no document. All reasons found for one operation are returned together. Reasons are listed block by block in tree order, and for one block in this order: position, placement, settings, binding. An invalid position, an unknown block, an unknown type and a private binding each end the checks for that block, since nothing further can be judged.
- **Settings validity.** Settings are valid when, after the schema's defaults are applied, they pass the type's settings schema. Stored settings are kept exactly as given (defaults are not written in). Each failing rule is one `InvalidSettings` refusal carrying the error's path.
- **`insert`.** The type must be registered (`UnknownType`). Placement is decided by `PlacementRulesInterface` against the parent block's type (`DisallowedPlacement`, with the decision's message). The new block gets a generated id not already in the document, the type's current version, the given settings, no extensions, bindings or children. `affectedIds` is the new id.
- **`insertBlocks`.** Every block in every subtree gets a new id (through the duplicator, reserving the document's ids and those already issued in this call). Each root is placed at `index`, `index + 1`, and so on. For every block of a registered type, placement against its parent (inside the subtree or at the position) and settings validity are checked; a block of an unregistered type is `UnknownType`. Nothing is inserted unless everything is valid. `affectedIds` is every new id in tree order.
- **`insertTemplate`.** The same as `insertBlocks` with the template's root blocks; the template is unchanged, and inserting it twice gives two sets of distinct ids.
- **`move`.** The position is read against the document with the block already removed. A position inside the block itself or one of its descendants is `MoveIntoSelf`. Placement is checked against the new parent. Ids do not change. Moving a block to where it already is is accepted and returns an equal document.
- **`remove`.** Removes the block and its descendants. `affectedIds` is empty.
- **`duplicate`.** The copy has new ids throughout and is placed directly after the original, with the same parent. Placement is not re-checked (the original is already there). `affectedIds` is the copy's ids.
- **`updateSettings`.** The block's type must be registered (`UnknownType`). The new settings replace the old ones entirely and must be valid. The block's stored type version is set to the type's current version. Bindings, extensions and children are kept.
- **`updateExtension`.** The namespace must name a registered decorator that applies to the block's type (`UnknownDecorator`). The settings must be valid against the decorator's schema (`InvalidSettings`, path prefixed `@namespace:`).
- **`setBinding`.** The path must address a declared setting of the block's type, or of an applicable decorator for a namespaced path (`InvalidBinding`). A path whose first segment is a private setting is `PrivateBinding`. The data source must be registered (`UnknownDataSource`). The binding's arguments must pass the source's `argumentsSchema()` (`InvalidBinding`). A fallback, when present, must pass the setting's subschema (`InvalidBinding`).
- **`clearBinding`.** Removes the binding at the path; a path with no binding is accepted and returns an equal document.

- [ ] **Step 1: Write the failing tests**

```php
// tests/Pest.php helper (guarded): editor(?callable $setup = null): array{0: DocumentEditorInterface, 1: BlockServiceContainer}
// helper refusals(EditResult $r): array => array_map(fn (EditRefusal $x) => $x->kind, $r->refusals)
$setup = function ($c) {
    types($c)->add(type('acme/form', ['children' => ['acme/field'], 'settings' => ['type' => 'object', 'properties' => [
        'title' => ['type' => 'string', 'maxLength' => 10], 'to' => ['type' => 'string', 'x-private' => true]]]]));
    types($c)->add(type('acme/field', ['parents' => ['acme/form'], 'version' => 2, 'settings' => ['type' => 'object', 'required' => ['key'],
        'properties' => ['key' => ['type' => 'string'], 'label' => ['type' => 'string', 'default' => 'Field']]]]));
    types($c)->add(type('acme/box', ['children' => true]));
    sources($c)->add(new ArrayDataSource('acme/values', ['k' => 'v'], ['type' => 'string']));
    $c->get(DecoratorRegistryInterface::class)->add(new Decorator('acme/vis', ['type' => 'object', 'properties' => ['hide' => ['type' => 'boolean']]], ['acme/box']));
};

test('insert adds a block with a new id at the position and leaves the input alone', function () use ($setup) {
    [$editor] = editor($setup);
    $document = doc(block('f', 'acme/form', [], block('n', 'acme/field', ['key' => 'name'])));
    $result = $editor->insert($document, BlockName::fromString('acme/field'), Position::inside('f', 0), ['key' => 'email']);
    $new = $result->affectedIds[0];
    expect($result->isAccepted())->toBeTrue()->and($result->document->tree->find('f')->children[0]->id)->toBe($new)
        ->and($new)->not->toBe('n')->and($result->document->tree->find($new)->settings)->toBe(['key' => 'email'])
        ->and($result->document->tree->find($new)->typeVersion)->toBe(2)->and($document->tree->ids())->toBe(['f', 'n']);
});
test('insert refuses what would break the document, with every reason', function () use ($setup) {
    [$editor] = editor($setup);
    $document = doc(block('f', 'acme/form'), block('b', 'acme/box'));
    $field = BlockName::fromString('acme/field');
    expect(refusals($editor->insert($document, $field, Position::root(0), ['key' => 'k'])))->toBe([EditRefusalKind::DisallowedPlacement])
        ->and(refusals($editor->insert($document, $field, Position::inside('b', 0), ['key' => 'k'])))->toBe([EditRefusalKind::DisallowedPlacement])
        ->and(refusals($editor->insert($document, $field, Position::inside('f', 0), [])))->toBe([EditRefusalKind::InvalidSettings])
        ->and(refusals($editor->insert($document, $field, Position::inside('f', 3), ['key' => 'k'])))->toBe([EditRefusalKind::InvalidPosition])
        ->and(refusals($editor->insert($document, $field, Position::inside('nope', 0), ['key' => 'k'])))->toBe([EditRefusalKind::InvalidPosition])
        ->and(refusals($editor->insert($document, BlockName::fromString('gone/away'), Position::root(0))))->toBe([EditRefusalKind::UnknownType])
        ->and(refusals($editor->insert($document, $field, Position::inside('b', 0), [])))->toBe([EditRefusalKind::DisallowedPlacement, EditRefusalKind::InvalidSettings])
        ->and($editor->insert($document, $field, Position::root(0))->document)->toBeNull();
});
test('pasted blocks always get new ids and are validated as a whole', function () use ($setup) {
    [$editor] = editor($setup);
    $document = doc(block('f', 'acme/form', [], block('n', 'acme/field', ['key' => 'name'])));
    $pasted = [block('f', 'acme/form', [], block('n', 'acme/field', ['key' => 'again'])), block('b', 'acme/box')];
    $result = $editor->insertBlocks($document, $pasted, Position::root(1));
    expect($result->isAccepted())->toBeTrue()->and(count($result->affectedIds))->toBe(3)
        ->and(array_intersect($result->affectedIds, ['f', 'n', 'b']))->toBe([])
        ->and(count(array_unique($result->document->tree->ids())))->toBe(5)
        ->and($pasted[0]->id)->toBe('f');
    $bad = [block('x', 'acme/box', [], block('y', 'acme/field', []))];
    expect(refusals($editor->insertBlocks($document, $bad, Position::root(0))))->toBe([EditRefusalKind::DisallowedPlacement, EditRefusalKind::InvalidSettings]);
});
test('a template can be inserted twice without id clashes and stays unchanged', function () use ($setup) {
    [$editor] = editor($setup);
    $template = new BlockTemplate('acme/contact', 'Contact', new BlockTree(block('t', 'acme/form', [], block('u', 'acme/field', ['key' => 'email']))));
    $once = $editor->insertTemplate(doc(), $template, Position::root(0));
    $twice = $editor->insertTemplate($once->document, $template, Position::root(1));
    expect($twice->isAccepted())->toBeTrue()->and(count(array_unique($twice->document->tree->ids())))->toBe(4)
        ->and($template->tree->ids())->toBe(['t', 'u']);
});
test('move keeps ids, checks the new parent, and cannot go inside itself', function () use ($setup) {
    [$editor] = editor($setup);
    $document = doc(block('a', 'acme/box', [], block('b', 'acme/box', [], block('c', 'acme/box'))), block('f', 'acme/form'));
    $moved = $editor->move($document, 'c', Position::root(0));
    expect($moved->document->tree->ids())->toBe(['c', 'a', 'b', 'f'])
        ->and(refusals($editor->move($document, 'a', Position::inside('c', 0))))->toBe([EditRefusalKind::MoveIntoSelf])
        ->and(refusals($editor->move($document, 'a', Position::inside('a', 0))))->toBe([EditRefusalKind::MoveIntoSelf])
        ->and(refusals($editor->move($document, 'c', Position::inside('f', 0))))->toBe([EditRefusalKind::DisallowedPlacement])
        ->and(refusals($editor->move($document, 'zz', Position::root(0))))->toBe([EditRefusalKind::UnknownBlock])
        ->and($editor->move($document, 'f', Position::root(1))->document->tree->ids())->toBe(['a', 'b', 'c', 'f']);
});
test('remove and duplicate', function () use ($setup) {
    [$editor] = editor($setup);
    $document = doc(block('a', 'acme/box', [], block('b', 'acme/box')), block('z', 'acme/box'));
    $copy = $editor->duplicate($document, 'a');
    expect($editor->remove($document, 'a')->document->tree->ids())->toBe(['z'])
        ->and(refusals($editor->remove($document, 'nope')))->toBe([EditRefusalKind::UnknownBlock])
        ->and(count($copy->affectedIds))->toBe(2)->and($copy->document->tree->roots()[1]->id)->toBe($copy->affectedIds[0])
        ->and($copy->document->tree->roots()[2]->id)->toBe('z')->and(count(array_unique($copy->document->tree->ids())))->toBe(5);
});
test('updateSettings replaces settings, validates them, and brings the block to the current type version', function () use ($setup) {
    [$editor] = editor($setup);
    $old = block('n', 'acme/field', ['key' => 'name'])->withBinding(SettingPath::fromString('label'), Binding::to('acme/values', ['key' => 'k']));
    $document = doc(block('f', 'acme/form', ['title' => 'Hi'], $old));
    $ok = $editor->updateSettings($document, 'n', ['key' => 'email']);
    $bad = $editor->updateSettings($document, 'f', ['title' => 'far too long a title']);
    expect($ok->document->tree->find('n')->settings)->toBe(['key' => 'email'])->and($ok->document->tree->find('n')->typeVersion)->toBe(2)
        ->and(array_keys($ok->document->tree->find('n')->bindings))->toBe(['label'])->and($ok->affectedIds)->toBe(['n'])
        ->and(refusals($bad))->toBe([EditRefusalKind::InvalidSettings])->and($bad->refusals[0]->path)->toBe('title')
        ->and(refusals($editor->updateSettings($document, 'n', [])))->toBe([EditRefusalKind::InvalidSettings]);
});
test('updateExtension needs a decorator that applies and settings that fit it', function () use ($setup) {
    [$editor] = editor($setup);
    $document = doc(block('b', 'acme/box'), block('f', 'acme/form'));
    expect($editor->updateExtension($document, 'b', 'acme/vis', ['hide' => true])->document->tree->find('b')->extensions)->toBe(['acme/vis' => ['hide' => true]])
        ->and(refusals($editor->updateExtension($document, 'f', 'acme/vis', ['hide' => true])))->toBe([EditRefusalKind::UnknownDecorator])
        ->and(refusals($editor->updateExtension($document, 'b', 'acme/nope', [])))->toBe([EditRefusalKind::UnknownDecorator])
        ->and($editor->updateExtension($document, 'b', 'acme/vis', ['hide' => 'yes'])->refusals[0]->path)->toBe('@acme/vis:hide');
});
test('setBinding refuses private settings, unknown sources, bad arguments, bad fallbacks and undeclared paths', function () use ($setup) {
    [$editor] = editor($setup);
    $document = doc(block('f', 'acme/form'));
    $path = fn (string $p) => SettingPath::fromString($p);
    $ok = $editor->setBinding($document, 'f', $path('title'), Binding::to('acme/values', ['key' => 'k'])->withFallback('Hello'));
    expect($ok->isAccepted())->toBeTrue()->and(array_keys($ok->document->tree->find('f')->bindings))->toBe(['title'])
        ->and(refusals($editor->setBinding($document, 'f', $path('to'), Binding::to('acme/values', ['key' => 'k']))))->toBe([EditRefusalKind::PrivateBinding])
        ->and(refusals($editor->setBinding($document, 'f', $path('title'), Binding::to('nope/source'))))->toBe([EditRefusalKind::UnknownDataSource])
        ->and(refusals($editor->setBinding($document, 'f', $path('title'), Binding::to('acme/values', ['key' => 5]))))->toBe([EditRefusalKind::InvalidBinding])
        ->and(refusals($editor->setBinding($document, 'f', $path('title'), Binding::to('acme/values', ['key' => 'k'])->withFallback('a fallback that is too long'))))->toBe([EditRefusalKind::InvalidBinding])
        ->and(refusals($editor->setBinding($document, 'f', $path('missing'), Binding::to('acme/values', ['key' => 'k']))))->toBe([EditRefusalKind::InvalidBinding])
        ->and($editor->clearBinding($ok->document, 'f', $path('title'))->document->tree->find('f')->bindings)->toBe([])
        ->and($editor->clearBinding($document, 'f', $path('title'))->isAccepted())->toBeTrue();
});
test('a document the editor accepts resolves without placement or settings issues', function () use ($setup) {
    [$editor, $c] = editor($setup);
    $result = $editor->insert(doc(), BlockName::fromString('acme/form'), Position::root(0), ['title' => 'Hi']);
    $result = $editor->insert($result->document, BlockName::fromString('acme/field'), Position::inside($result->affectedIds[0], 0), ['key' => 'email']);
    expect($c->get(DocumentResolverInterface::class)->resolve($result->document, Mode::Front)->hasIssues())->toBeFalse();
});
```

- [ ] **Step 2: Run `vendor/bin/pest --colors=never` in `block-app`.** Expected: FAIL, classes not found.
- [ ] **Step 3: Implement** the value objects, contract, editor and binding.
- [ ] **Step 4: Run `vendor/bin/pest --colors=never` in `block-domain` and `block-app`.** Expected: PASS.
- [ ] **Step 5: Commit** `feat(block): add validated editing operations`.

---

### Task 5: Preview and the admin placeholder

**Files:**
- Create in `block-domain/src/`: `ValueObjects/BlockPreview.php`, `Contracts/{BlockPreviewInterface,UnresolvedBlockOutputInterface}.php`
- Create in `block-app/src/Building/`: `BlockPreviewer.php`, `PlaceholderOutput.php`
- Modify: `block-app/src/Output/DocumentRenderer.php`, `block-app/src/AppServiceProvider.php`, the repository `README.md` (a short "For builders" section naming the contracts a builder uses)
- Test: `block-app/tests/Building/BlockPreviewerTest.php`, additions to `block-app/tests/Output/DocumentRendererTest.php` and `tests/AppServiceProviderTest.php`

**Interfaces:**
- `UnresolvedBlockOutputInterface`: `render(ResolvedBlock $block, RenderRequest $request, OutputEscaperInterface $escaper): string` — what to draw, in admin mode only, for a block that produced no view, so a builder's canvas has something to select.
- `PlaceholderOutput implements UnresolvedBlockOutputInterface`: draws `<div class="block-placeholder" data-block-id="{id}" data-block-type="{type}"></div>` with both values written through the escaper's attribute bag.
- `DocumentRenderer` takes `?UnresolvedBlockOutputInterface $unresolved = null` as a new last constructor parameter. In `Mode::Admin`, when it is not null, a block with no view yields the placeholder's output instead of `''` (a placeholder that throws records `OutputFailed` and yields `''`). In `Mode::Front` nothing changes: such a block yields `''`. The provider binds `PlaceholderOutput` under the contract and passes it to the renderer.
- `BlockPreview` (readonly): `__construct(string $blockId, ?ResolvedBlock $block, string $output, array $issues, array $assets)`. `$block` is null when the block exists in the document but was not reached by resolving (its parent stopped, or it was misplaced). `$issues` are the issues of the block and its descendants, from resolving and drawing.
- `BlockPreviewInterface`: `preview(Document $document, string $blockId, RenderRequest $request, ?Context $context = null): ?BlockPreview` — null when the document has no block with that id. The whole document is resolved (so the block gets the context its ancestors provide) and only that block and its descendants are drawn.
- `BlockPreviewer::__construct(DocumentResolverInterface $resolver, DocumentRendererInterface $renderer)`. It draws by calling `renderResolved()` with a `ResolvedDocument` whose only root is the block and whose issues are that subtree's.

- [ ] **Step 1: Write the failing tests**

```php
// tests/Pest.php helper (guarded): previewer(?callable $setup = null): array{0: BlockPreviewInterface, 1: BlockServiceContainer}
$setup = function ($c) {
    types($c)->add(type('acme/form', ['children' => ['acme/field'], 'provides' => ['acme/form-id' => 'formId'],
        'settings' => ['type' => 'object', 'properties' => ['formId' => ['type' => 'string', 'default' => 'f1']]]]));
    types($c)->add(type('acme/field', ['parents' => ['acme/form'], 'uses' => ['acme/form-id'], 'assets' => ['field'],
        'settings' => ['type' => 'object', 'required' => ['key'], 'properties' => ['key' => ['type' => 'string']]]]));
    outputs($c)->set(BlockName::fromString('acme/form'), tag('form'));
    outputs($c)->set(BlockName::fromString('acme/field'), new class implements BlockOutputInterface {
        public function render(RenderInput $input): string
        {
            return '<input name="' . $input->escaper->attribute((string) $input->view->values['key']) . '" form="' . $input->escaper->attribute((string) $input->context->get('acme/form-id')) . '">';
        }
    });
    outputs($c)->set(BlockName::fromString('acme/field'), fixedOutput('[editing field]'), Mode::Admin);
};
$document = fn () => doc(block('f', 'acme/form', [], block('n', 'acme/field', ['key' => 'name']), block('bad', 'acme/field', [])));

test('previews one block with the context its ancestors provide', function () use ($setup, $document) {
    [$previewer] = previewer($setup);
    $preview = $previewer->preview($document(), 'n', new RenderRequest(Mode::Front));
    expect($preview->output)->toBe('<input name="name" form="f1">')->and($preview->issues)->toBe([])
        ->and($preview->assets)->toBe(['field'])->and($preview->block->view->values)->toBe(['key' => 'name'])
        ->and($preview->blockId)->toBe('n');
});
test('admin preview uses the admin output and carries the admin context', function () use ($setup, $document) {
    [$previewer] = previewer($setup);
    $preview = $previewer->preview($document(), 'n', new RenderRequest(Mode::Admin));
    expect($preview->output)->toBe('[editing field]')->and($preview->block->admin)->toBeInstanceOf(AdminContext::class);
});
test('a broken block previews as a placeholder in admin mode and as nothing for a visitor', function () use ($setup, $document) {
    [$previewer] = previewer($setup);
    $admin = $previewer->preview($document(), 'bad', new RenderRequest(Mode::Admin));
    $front = $previewer->preview($document(), 'bad', new RenderRequest(Mode::Front));
    expect($admin->output)->toBe('<div class="block-placeholder" data-block-id="bad" data-block-type="acme/field"></div>')
        ->and(kinds($admin->issues))->toBe([IssueKind::InvalidSettings])->and($admin->block->admin->issues)->toHaveCount(1)
        ->and($front->output)->toBe('')->and(kinds($front->issues))->toBe([IssueKind::InvalidSettings]);
});
test('previewing a parent draws its children and reports only that subtree', function () use ($setup) {
    [$previewer] = previewer($setup);
    $document = doc(block('x', 'gone/away'), block('f', 'acme/form', [], block('n', 'acme/field', ['key' => 'name'])));
    $preview = $previewer->preview($document, 'f', new RenderRequest(Mode::Front));
    expect($preview->output)->toBe('<form><input name="name" form="f1"></form>')->and($preview->issues)->toBe([]);
});
test('a missing id gives no preview, and a block resolving never reached gives one without a resolved block', function () use ($setup) {
    [$previewer] = previewer($setup);
    $document = doc(block('f', 'acme/form', ['formId' => 5], block('n', 'acme/field', ['key' => 'name'])));
    expect($previewer->preview($document, 'nope', new RenderRequest()))->toBeNull();
    $unreached = $previewer->preview($document, 'n', new RenderRequest(Mode::Admin));
    expect($unreached->block)->toBeNull()->and($unreached->output)->toBe('')->and($unreached->issues)->toBe([]);
});
// DocumentRendererTest additions
test('in admin mode an unresolved block is drawn as a placeholder, and for a visitor it is not', function () {
    [$renderer] = renderer(function ($c) { types($c)->add(type('acme/ok')); outputs($c)->set(BlockName::fromString('acme/ok'), fixedOutput('ok')); });
    $document = doc(block('1', 'gone/away'), block('2', 'acme/ok'));
    expect($renderer->render($document, new RenderRequest(Mode::Admin))->output)
        ->toBe('<div class="block-placeholder" data-block-id="1" data-block-type="gone/away"></div>ok')
        ->and($renderer->render($document, new RenderRequest(Mode::Front))->output)->toBe('ok');
});
test('a block id that holds markup is escaped in the placeholder', function () {
    [$renderer] = renderer();
    $out = $renderer->render(doc(block('"><script>', 'gone/away')), new RenderRequest(Mode::Admin))->output;
    expect($out)->not->toContain('<script>')->toContain('data-block-id="&quot;&gt;&lt;script&gt;"');
});
```

Also extend the existing "every application contract resolves" test in `tests/AppServiceProviderTest.php` with every contract bound since part 1b: `OutputEscaperRegistryInterface`, `BlockOutputRegistryInterface`, `DocumentRendererInterface`, `BlockPackageLoaderInterface`, `PlacementRulesInterface`, `BlockCatalogueInterface`, `BlockTemplateRegistryInterface`, `DocumentEditorInterface`, `BlockPreviewInterface`, `UnresolvedBlockOutputInterface`; and add a test that replacing `UnresolvedBlockOutputInterface` changes what the renderer draws for an unresolved block.

- [ ] **Step 2: Run `vendor/bin/pest --colors=never` in `block-app`.** Expected: FAIL.
- [ ] **Step 3: Implement** the value object, contracts, previewer, placeholder, renderer change, bindings and README section.
- [ ] **Step 4: Run `vendor/bin/pest --colors=never` in `block-domain`, `block-app`, `block-infra-memory` and `block-example`.** Expected: PASS in each.
- [ ] **Step 5: Commit** `feat(block): add block preview and the admin placeholder`.

---

## Out of scope for this plan

A builder UI; exposing these contracts over HTTP (phase 2); permission checks; undo and redo; collaborative editing; per-block output in a full-document render; storing documents or templates anywhere but memory; the presentation vocabulary, conditions, loops, structural blocks and the WordPress adapter. The remaining "Deferred from part 1a" and "Deferred from part 1b" items stay deferred, except the resolver's own placement check, which Task 1 replaces with the placement contract.
