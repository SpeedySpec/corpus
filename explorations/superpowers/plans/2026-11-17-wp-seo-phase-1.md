# wp-seo Phase 1 (contracts and head core) Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Ship the `wp-seo` contracts package and standalone, replaceable implementations that resolve and render a page's title, meta description, robots directives and canonical URL.

**Architecture:** Domain-Driven Design layers as separate Composer packages under `packages/wp-seo/`. `wp-seo-domain` holds every contract plus value objects and a service container; `wp-seo-app` holds framework-free use cases; `wp-seo-infra-memory` and `wp-seo-infra-wp` hold host adapters; `wp-seo-legacy` holds the existing WordPress functions as wrappers. Every contract is bound individually in the container so it can be replaced alone, and each layer is a separate package so it can be replaced whole.

**Tech Stack:** PHP 8.4, Composer path repositories, Pest 3/4.

**Spec:** `docs/superpowers/specs/2026-11-17-seo-api-design.md`

## Global Constraints

- PHP `>=8.4`; every PHP file starts with `declare(strict_types=1);`; 4-space indentation; opening braces on a new line; docblocks follow `.claude/rules/coding-style.md` (summary line, "why" not "how", no `@return void`, `@since 0.1.0`).
- Code moved verbatim from `wp-theme-legacy` keeps its existing tab formatting.
- `wp-seo-domain` requires only PHP and calls no WordPress function. No class in it is `final`.
- `wp-seo-app` requires only `wp-seo-domain`. Infrastructure packages never require each other.
- `wp-seo-legacy` never names a class from `wp-seo-app` or any infra package in `src/` or `functions/`.
- Namespaces: `SpeedySpec\WP\Seo\Domain\`, `...\App\`, `...\Infra\Memory\`, `...\Infra\WP\`, `...\Legacy\`, `...\ContractTests\`.
- Composer names: `speedyspec/wp-seo-domain`, `-app`, `-infra-memory`, `-infra-wp`, `-legacy`, `-contract-tests`. Each has `"version": "0.1.0"`, license `GPL-2.0-or-later`, and the same `archive.exclude`, `config.allow-plugins` and `phpunit.xml` as `packages/speedyspec-wp-hook-domain`.
- Sibling packages are linked with `"repositories": [{"type": "path", "url": "../*"}]` and required as `"^0.1"`.
- Virtual packages: `wp-seo-app` has `"provide": {"speedyspec/wp-seo-app-implementation": "1.0"}`; both infra packages have `"provide": {"speedyspec/wp-seo-infra-implementation": "1.0"}`.
- Tests are Pest, files end `Test.php`, run with `composer install && vendor/bin/pest` inside the package directory.
- Standalone only: do not wire packages into `src/`, the root `composer.json`, or `tests/phpunit`.
- No length limits on title or description. No meta keywords. Robots directive names are an open set.
- Work on branch `feat/wp-seo-phase-1`, not `trunk`. Never commit `vendor/` or `composer.lock` inside packages.

## Review Focus

Each line has a pinning test in the task named.

1. **Title or description already containing HTML entities** (WordPress titles arrive as `Search Results for &#8220;x&#8221;`). Expected: rendered once, never double-encoded. Task 6 (renderer) and Task 8 (legacy title printed unescaped, as today).
2. **A legacy filter returning garbage** (`get_canonical_url` returning `false`, `''` or a relative path; `wp_robots` returning a non-array or keys such as `NOINDEX` or `bad key`). Expected: no canonical tag, invalid directives dropped, no exception reaches the page. Task 7.
3. **A legacy function called when no implementation is bound.** Expected: `wp_get_document_title()` returns `''`, output functions print nothing, no fatal. Task 8.
4. **Replacing a contract after it was already resolved.** Expected: `add()` and `remove()` throw `ServiceAlreadyResolvedException` instead of silently keeping the old instance. Task 3.
5. **Whitespace-only or tag-only content summary.** Expected: no description tag at all, not an empty one. Task 5 and Task 6.

---

## File Structure

```
packages/wp-seo/
├── wp-seo-domain/src/
│   ├── SeoServiceContainer.php
│   ├── Contracts/            one interface per file (Task 4 lists them)
│   ├── Entities/HeadDocument.php
│   ├── ValueObjects/         Url, PageKind, PageContext, RobotsDirectives, MetaAttribute, MetaTag, LinkTag, SeoOverrides
│   └── Exceptions/           InvalidUrlException, InvalidRobotsDirectiveException,
│                             ServiceNotRegisteredException, ServiceAlreadyResolvedException
├── wp-seo-contract-tests/src/   one `*Contract.php` function file per contract
├── wp-seo-infra-memory/src/     InMemory*/Array*/Static*/Recording* adapters, MemoryServiceProvider.php
├── wp-seo-app/src/
│   ├── UseCases/             ResolveCanonicalUrl, ResolveTitle, ResolveDescription, ResolveRobotsDirectives, BuildHeadDocument
│   ├── Head/                 HeadContributorRegistry, HtmlHeadRenderer, Contributors/{Title,Description,Robots,Canonical}Contributor
│   └── AppServiceProvider.php
├── wp-seo-infra-wp/src/
│   ├── Adapters/             WordPressPageContextProvider, WordPressPermalinkSource, WordPressSiteInfoProvider,
│   │                         WordPressContentSource, WordPressEventDispatcher, WordPressResponseHeaderEmitter,
│   │                         NullSeoOverridesRepository
│   ├── Resolvers/            WordPressTitleResolver, WordPressRobotsDirectivesResolver, WordPressCanonicalUrlResolver
│   └── WordPressServiceProvider.php
└── wp-seo-legacy/
    ├── functions/seo.php     moved functions as wrappers, plus wp_seo_render_head()
    └── src/HeadAccess.php    the one place legacy code touches the container
```

---

### Task 1: Domain package with `Url` and `RobotsDirectives`

**Files:**
- Create: `packages/wp-seo/wp-seo-domain/{composer.json,phpunit.xml,tests/Pest.php}`
- Create: `src/ValueObjects/Url.php`, `src/ValueObjects/RobotsDirectives.php`, `src/Exceptions/InvalidUrlException.php`, `src/Exceptions/InvalidRobotsDirectiveException.php`
- Test: `tests/ValueObjects/UrlTest.php`, `tests/ValueObjects/RobotsDirectivesTest.php`

**Interfaces:**
- Produces `Url`: `static fromString(string $value): self` (throws `InvalidUrlException` unless scheme is `http` or `https` and a host is present), `static tryFromString(mixed $value): ?self` (null instead of throwing, null for non-strings), `toString(): string`, `scheme(): string`, `withScheme(string $scheme): self`, `equals(self $other): bool`.
- Produces `RobotsDirectives` (immutable, insertion-ordered map of directive name to `true|string`): `static none(): self`, `static fromArray(mixed $directives): self`, `with(string $name, bool|string $value = true): self`, `without(string $name): self`, `has(string $name): bool`, `get(string $name): true|string|null`, `merge(self $other): self` (other wins), `isEmpty(): bool`, `toArray(): array`, `toString(): string`.
- Directive names match `/^[a-z][a-z0-9_-]*$/` after lower-casing. `with()` throws `InvalidRobotsDirectiveException` for an invalid name; `fromArray()` silently drops invalid names, drops falsy values, keeps string values, turns other truthy values into `true`, and returns `none()` for a non-array.

- [ ] **Step 1: Scaffold the package** (composer.json, phpunit.xml, `tests/Pest.php` containing only `declare(strict_types=1);`), run `composer install`. Expected: Pest installed, `vendor/bin/pest` reports no tests.

- [ ] **Step 2: Write the failing tests**

```php
// tests/ValueObjects/UrlTest.php
use SpeedySpec\WP\Seo\Domain\Exceptions\InvalidUrlException;
use SpeedySpec\WP\Seo\Domain\ValueObjects\Url;

test('accepts absolute http and https urls', function () {
    expect(Url::fromString('https://example.com/a?b=1')->toString())->toBe('https://example.com/a?b=1');
});
test('rejects relative, scheme-less and non-http urls', function (string $value) {
    expect(fn () => Url::fromString($value))->toThrow(InvalidUrlException::class);
})->with(['/path', 'example.com/a', '//example.com/a', 'ftp://example.com', 'javascript:alert(1)', '']);
test('tryFromString returns null for garbage', function (mixed $value) {
    expect(Url::tryFromString($value))->toBeNull();
})->with([false, null, '', '/relative', 42]);
test('withScheme swaps only the scheme', function () {
    expect(Url::fromString('http://example.com/a')->withScheme('https')->toString())->toBe('https://example.com/a');
});
```

```php
// tests/ValueObjects/RobotsDirectivesTest.php
use SpeedySpec\WP\Seo\Domain\Exceptions\InvalidRobotsDirectiveException;
use SpeedySpec\WP\Seo\Domain\ValueObjects\RobotsDirectives;

test('renders flags and valued directives in insertion order', function () {
    $d = RobotsDirectives::none()->with('noindex')->with('max-image-preview', 'large');
    expect($d->toString())->toBe('noindex, max-image-preview:large');
});
test('noarchive, nocache and nosnippet stay distinct', function () {
    $d = RobotsDirectives::none()->with('noarchive')->with('nocache');
    expect($d->has('noarchive'))->toBeTrue()->and($d->has('nocache'))->toBeTrue()->and($d->has('nosnippet'))->toBeFalse();
});
test('accepts directives it has never heard of', function () {
    expect(RobotsDirectives::none()->with('some-future-directive')->toString())->toBe('some-future-directive');
});
test('fromArray follows WordPress truthiness and drops invalid names', function () {
    $d = RobotsDirectives::fromArray(['NOINDEX' => true, 'follow' => 0, 'max-snippet' => '-1', 'bad key' => true, 5 => true]);
    expect($d->toArray())->toBe(['noindex' => true, 'max-snippet' => '-1']);
});
test('fromArray returns none for a non-array', function () {
    expect(RobotsDirectives::fromArray('noindex')->isEmpty())->toBeTrue();
});
test('with rejects an invalid name', function () {
    expect(fn () => RobotsDirectives::none()->with('bad key'))->toThrow(InvalidRobotsDirectiveException::class);
});
test('merge lets the other set win and is immutable', function () {
    $a = RobotsDirectives::none()->with('max-snippet', '50');
    $b = $a->merge(RobotsDirectives::none()->with('max-snippet', '-1'));
    expect($b->get('max-snippet'))->toBe('-1')->and($a->get('max-snippet'))->toBe('50');
});
```

- [ ] **Step 3: Run `vendor/bin/pest`.** Expected: FAIL, classes not found.
- [ ] **Step 4: Implement the two value objects and two exceptions** to the signatures above. `Url` parses with `parse_url`.
- [ ] **Step 5: Run `vendor/bin/pest`.** Expected: PASS.
- [ ] **Step 6: Commit** `feat(wp-seo-domain): add Url and RobotsDirectives value objects`.

---

### Task 2: Page and head value objects

**Files:**
- Create in `wp-seo-domain/src/`: `ValueObjects/PageKind.php`, `ValueObjects/PageContext.php`, `ValueObjects/MetaAttribute.php`, `ValueObjects/MetaTag.php`, `ValueObjects/LinkTag.php`, `ValueObjects/SeoOverrides.php`, `Entities/HeadDocument.php`
- Test: `tests/ValueObjects/PageContextTest.php`, `tests/Entities/HeadDocumentTest.php`

**Interfaces:**
- Consumes: `Url`, `RobotsDirectives`.
- Produces `enum PageKind: string` with cases `Singular`, `FrontPage`, `PostsIndex`, `TermArchive`, `AuthorArchive`, `DateArchive`, `PostTypeArchive`, `Search`, `NotFound`, `Other` (values are the snake_case names).
- Produces `PageContext` (readonly): `__construct(PageKind $kind, ?string $objectType = null, ?int $objectId = null, int $page = 1, ?string $locale = null, int $status = 200)`; public readonly properties of the same names; `isSingular(): bool`; `equals(self $other): bool`. `$page < 1` is stored as `1`.
- Produces `enum MetaAttribute: string { case Name = 'name'; case Property = 'property'; }`.
- Produces `MetaTag` (readonly): `__construct(MetaAttribute $attribute, string $key, string $content)`.
- Produces `LinkTag` (readonly): `__construct(string $rel, string $href, array $attributes = [])` where `$attributes` is `array<string, string>`.
- Produces `SeoOverrides` (readonly): `__construct(?string $title = null, ?string $description = null, ?Url $canonical = null, ?RobotsDirectives $robots = null, ?Url $image = null)`, `static none(): self`, `isEmpty(): bool`.
- Produces `HeadDocument` (mutable): `title(): ?string`, `setTitle(?string $title): void`, `meta(string $key): ?MetaTag`, `setMeta(MetaTag $tag): void` (replaces same key), `removeMeta(string $key): void`, `metas(): array` (list, insertion order), `link(string $rel): ?LinkTag`, `setLink(LinkTag $tag): void` (replaces same rel), `removeLink(string $rel): void`, `links(): array`.

- [ ] **Step 1: Write the failing tests**

```php
// tests/ValueObjects/PageContextTest.php
test('only the Singular kind is singular', function () {
    expect((new PageContext(PageKind::Singular, 'post', 7))->isSingular())->toBeTrue()
        ->and((new PageContext(PageKind::Search))->isSingular())->toBeFalse();
});
test('page below one is normalised to one', function () {
    expect((new PageContext(PageKind::PostsIndex, page: 0))->page)->toBe(1);
});
test('equals compares every field', function () {
    $a = new PageContext(PageKind::Singular, 'post', 7, 2);
    expect($a->equals(new PageContext(PageKind::Singular, 'post', 7, 2)))->toBeTrue()
        ->and($a->equals(new PageContext(PageKind::Singular, 'post', 7, 3)))->toBeFalse();
});

// tests/Entities/HeadDocumentTest.php
test('setMeta replaces a tag with the same key and keeps order', function () {
    $doc = new HeadDocument();
    $doc->setMeta(new MetaTag(MetaAttribute::Name, 'description', 'one'));
    $doc->setMeta(new MetaTag(MetaAttribute::Property, 'og:title', 'T'));
    $doc->setMeta(new MetaTag(MetaAttribute::Name, 'description', 'two'));
    expect(array_map(fn ($t) => $t->key, $doc->metas()))->toBe(['description', 'og:title'])
        ->and($doc->meta('description')->content)->toBe('two');
});
test('removeMeta and removeLink ignore unknown keys', function () {
    $doc = new HeadDocument();
    $doc->removeMeta('nope');
    $doc->removeLink('nope');
    expect($doc->metas())->toBe([])->and($doc->links())->toBe([])->and($doc->title())->toBeNull();
});
```

- [ ] **Step 2: Run `vendor/bin/pest`.** Expected: FAIL, classes not found.
- [ ] **Step 3: Implement the classes** to the signatures above.
- [ ] **Step 4: Run `vendor/bin/pest`.** Expected: PASS.
- [ ] **Step 5: Commit** `feat(wp-seo-domain): add page context and head document model`.

---

### Task 3: `SeoServiceContainer`

**Files:**
- Create: `wp-seo-domain/src/SeoServiceContainer.php`, `src/Exceptions/ServiceNotRegisteredException.php`, `src/Exceptions/ServiceAlreadyResolvedException.php`
- Test: `tests/SeoServiceContainerTest.php`

**Interfaces:**
- Produces `SeoServiceContainer` (not `final`, public constructor): `add(string $contract, callable $factory): static` (factory receives the container), `remove(string $contract): static`, `extend(string $contract, callable $decorator): static` (decorator receives the inner instance and the container, and returns the replacement), `has(string $contract): bool`, `get(string $contract): object`, `static shared(): self`, `static resetShared(): void`.
- `get()` throws `ServiceNotRegisteredException` when unbound and caches the instance. `add()`, `remove()` and `extend()` throw `ServiceAlreadyResolvedException` once that contract has been resolved. `extend()` throws `ServiceNotRegisteredException` when nothing is bound. Both exceptions extend `\RuntimeException`.
- Do not copy `HookServiceContainer` or `OptionServiceContainer`; they are `final`, lack `has()` and keep stale instances.

- [ ] **Step 1: Write the failing tests**

```php
test('get resolves lazily, passes the container, and caches', function () {
    $c = new SeoServiceContainer();
    $calls = 0;
    $c->add(stdClass::class, function (SeoServiceContainer $inner) use (&$calls, $c) {
        $calls++;
        expect($inner)->toBe($c);
        return new stdClass();
    });
    expect($calls)->toBe(0)->and($c->get(stdClass::class))->toBe($c->get(stdClass::class))->and($calls)->toBe(1);
});
test('a later add replaces an unresolved binding', function () {
    $c = (new SeoServiceContainer())->add('x', fn () => new ArrayObject(['first']))->add('x', fn () => new ArrayObject(['second']));
    expect($c->get('x')[0])->toBe('second');
});
test('add and remove throw once the contract was resolved', function () {
    $c = (new SeoServiceContainer())->add('x', fn () => new stdClass());
    $c->get('x');
    expect(fn () => $c->add('x', fn () => new stdClass()))->toThrow(ServiceAlreadyResolvedException::class)
        ->and(fn () => $c->remove('x'))->toThrow(ServiceAlreadyResolvedException::class);
});
test('extend decorates the existing binding lazily', function () {
    $c = (new SeoServiceContainer())->add('x', fn () => new ArrayObject(['inner']));
    $c->extend('x', fn (ArrayObject $inner, SeoServiceContainer $container) => new ArrayObject([$inner[0], 'outer']));
    expect($c->get('x')->getArrayCopy())->toBe(['inner', 'outer']);
});
test('extend throws when nothing is bound', function () {
    expect(fn () => (new SeoServiceContainer())->extend('x', fn ($inner) => $inner))->toThrow(ServiceNotRegisteredException::class);
});
test('get throws for an unbound contract and has reports it', function () {
    $c = new SeoServiceContainer();
    expect($c->has('x'))->toBeFalse()->and(fn () => $c->get('x'))->toThrow(ServiceNotRegisteredException::class);
});
test('shared returns one instance until reset', function () {
    $a = SeoServiceContainer::shared();
    expect(SeoServiceContainer::shared())->toBe($a);
    SeoServiceContainer::resetShared();
    expect(SeoServiceContainer::shared())->not->toBe($a);
});
```

- [ ] **Step 2: Run `vendor/bin/pest tests/SeoServiceContainerTest.php`.** Expected: FAIL.
- [ ] **Step 3: Implement the container and exceptions.**
- [ ] **Step 4: Run `vendor/bin/pest`.** Expected: PASS.
- [ ] **Step 5: Commit** `feat(wp-seo-domain): add per-contract service container`.

---

### Task 4: Contracts, contract tests, and in-memory infrastructure

The contracts have no behaviour of their own, so they land with the package that proves them: the shared contract tests, run against the in-memory adapters.

**Files:**
- Create: `wp-seo-domain/src/Contracts/*.php` (14 interfaces below)
- Create: `packages/wp-seo/wp-seo-contract-tests/{composer.json,src/*.php}` (requires `speedyspec/wp-seo-domain` and `pestphp/pest` as real dependencies; `src/*.php` loaded through `autoload.files`)
- Create: `packages/wp-seo/wp-seo-infra-memory/{composer.json,phpunit.xml,tests/Pest.php,src/*.php}` (requires domain; require-dev contract-tests)
- Test: `wp-seo-infra-memory/tests/ContractsTest.php`, `tests/MemoryServiceProviderTest.php`

**Interfaces (all in `SpeedySpec\WP\Seo\Domain\Contracts`):**

| Interface | Methods |
|---|---|
| `PageContextProviderInterface` | `current(): PageContext` |
| `PermalinkSourceInterface` | `permalinkFor(PageContext $context): ?Url` |
| `SiteInfoProviderInterface` | `name(): string`, `tagline(): string`, `isPublic(): bool`, `usesHttps(): bool` |
| `ContentSourceInterface` | `titleFor(PageContext $context): ?string`, `summaryFor(PageContext $context): ?string` |
| `SeoOverridesRepositoryInterface` | `find(string $objectType, int $objectId): SeoOverrides`, `save(string $objectType, int $objectId, SeoOverrides $overrides): void`, `delete(string $objectType, int $objectId): void` |
| `CanonicalUrlResolverInterface` | `resolve(PageContext $context): ?Url` |
| `TitleResolverInterface` | `resolve(PageContext $context): string` |
| `DescriptionResolverInterface` | `resolve(PageContext $context): ?string` |
| `RobotsDirectivesResolverInterface` | `resolve(PageContext $context): RobotsDirectives` |
| `HeadContributorInterface` | `contribute(HeadDocument $document, PageContext $context): void` |
| `HeadContributorRegistryInterface` | `add(string $id, HeadContributorInterface $contributor, int $priority = 10): void`, `remove(string $id): void`, `has(string $id): bool`, `all(): array` (list ordered by priority, then insertion) |
| `HeadDocumentBuilderInterface` | `build(PageContext $context): HeadDocument` |
| `HeadRendererInterface` | `renderTitle(string $title): string`, `renderMeta(MetaTag $tag): string`, `renderLink(LinkTag $tag): string`, `render(HeadDocument $document): string` |
| `ResponseHeaderEmitterInterface` | `emit(string $name, string $value, bool $replace = true): void` |
| `EventDispatcherInterface` | `dispatch(object $event): void`, `filter(string $name, mixed $value, mixed ...$arguments): mixed` |

`SiteInfoProviderInterface`, `ContentSourceInterface` and `HeadDocumentBuilderInterface` are additions to the spec's list; the framework-free resolvers and the "built once per request" rule need them.

**Contract tests** (namespace `SpeedySpec\WP\Seo\ContractTests`, each a function taking `Closure $make` and registering a Pest `describe()` block):

| Function | Invariants asserted |
|---|---|
| `seoOverridesRepositoryContract(Closure $make)` | unknown object returns an empty `SeoOverrides`; `save` then `find` round-trips title, description, canonical and robots; `delete` then `find` is empty; objects of different type with the same id do not collide |
| `headContributorRegistryContract(Closure $make)` | `all()` is ordered by priority then insertion; `add` with an existing id replaces it; `remove` of an unknown id is a no-op; `has` reflects both |
| `eventDispatcherContract(Closure $make)` | `filter` with no listener returns the value unchanged; `dispatch` with no listener does not throw |
| `resolverContract(Closure $make, string $interface)` | calling `resolve()` twice with equal `PageContext` values returns equal results, for each `PageKind` case, without throwing |

**In-memory adapters** (namespace `SpeedySpec\WP\Seo\Infra\Memory`):
- `InMemorySeoOverridesRepository`.
- `StaticPageContextProvider`: `__construct(?PageContext $context = null)` (default `PageKind::Other`), `set(PageContext $context): void`.
- `ArrayPermalinkSource`: `set(PageContext $context, Url $url): void`; unknown context returns null; matches with `PageContext::equals`.
- `ArraySiteInfoProvider`: `__construct(string $name = '', string $tagline = '', bool $isPublic = true, bool $usesHttps = true)`.
- `ArrayContentSource`: `set(PageContext $context, ?string $title, ?string $summary): void`.
- `RecordingResponseHeaderEmitter`: `emitted(): array` (list of `[name, value, replace]`).
- `InMemoryEventDispatcher`: `addFilter(string $name, callable $callback): void`, `listen(string $eventClass, callable $listener): void`, `dispatched(): array`.
- `MemoryServiceProvider::register(SeoServiceContainer $container): void` binds the seven data and output contracts above, one `add()` each.

- [ ] **Step 1: Write the interfaces** in `wp-seo-domain/src/Contracts/`. Run `vendor/bin/pest` in `wp-seo-domain`. Expected: PASS (unchanged count).
- [ ] **Step 2: Write the contract-test functions** in `wp-seo-contract-tests/src/` with the invariants in the table.
- [ ] **Step 3: Write the failing tests in `wp-seo-infra-memory`**

```php
// tests/ContractsTest.php
use function SpeedySpec\WP\Seo\ContractTests\{eventDispatcherContract, seoOverridesRepositoryContract};

seoOverridesRepositoryContract(fn () => new InMemorySeoOverridesRepository());
eventDispatcherContract(fn () => new InMemoryEventDispatcher());

// tests/MemoryServiceProviderTest.php
test('binds each data contract separately', function () {
    $c = new SeoServiceContainer();
    (new MemoryServiceProvider())->register($c);
    foreach ([PageContextProviderInterface::class, PermalinkSourceInterface::class, SiteInfoProviderInterface::class,
        ContentSourceInterface::class, SeoOverridesRepositoryInterface::class, ResponseHeaderEmitterInterface::class,
        EventDispatcherInterface::class] as $contract) {
        expect($c->get($contract))->toBeInstanceOf($contract);
    }
});
test('one binding can be replaced without touching the rest', function () {
    $c = new SeoServiceContainer();
    (new MemoryServiceProvider())->register($c);
    $mine = new ArraySiteInfoProvider(name: 'Mine');
    $c->add(SiteInfoProviderInterface::class, fn () => $mine);
    expect($c->get(SiteInfoProviderInterface::class))->toBe($mine)
        ->and($c->get(EventDispatcherInterface::class))->toBeInstanceOf(InMemoryEventDispatcher::class);
});
test('filters run in registration order with extra arguments', function () {
    $d = new InMemoryEventDispatcher();
    $d->addFilter('f', fn ($v, $x) => $v . $x);
    $d->addFilter('f', fn ($v) => strtoupper($v));
    expect($d->filter('f', 'a', 'b'))->toBe('AB');
});
```

- [ ] **Step 4: Run `composer install && vendor/bin/pest` in `wp-seo-infra-memory`.** Expected: FAIL, classes not found.
- [ ] **Step 5: Implement the adapters and provider.**
- [ ] **Step 6: Run `vendor/bin/pest`.** Expected: PASS.
- [ ] **Step 7: Commit** `feat(wp-seo): add contracts, contract tests and in-memory infrastructure`.

---

### Task 5: Application resolvers

**Files:**
- Create: `packages/wp-seo/wp-seo-app/{composer.json,phpunit.xml,tests/Pest.php}` (requires domain; require-dev infra-memory and contract-tests)
- Create: `src/UseCases/{ResolveCanonicalUrl,ResolveTitle,ResolveDescription,ResolveRobotsDirectives}.php`
- Test: `tests/UseCases/*Test.php`, `tests/ContractsTest.php`

**Interfaces:**
- Consumes: Task 4 contracts and in-memory adapters.
- Produces (namespace `SpeedySpec\WP\Seo\App\UseCases`):
  - `ResolveCanonicalUrl implements CanonicalUrlResolverInterface`: `__construct(PermalinkSourceInterface $permalinks, SeoOverridesRepositoryInterface $overrides, SiteInfoProviderInterface $site)`.
  - `ResolveTitle implements TitleResolverInterface`: `__construct(ContentSourceInterface $content, SeoOverridesRepositoryInterface $overrides, SiteInfoProviderInterface $site, string $separator = '-', string $pageLabel = 'Page %s')`.
  - `ResolveDescription implements DescriptionResolverInterface`: `__construct(ContentSourceInterface $content, SeoOverridesRepositoryInterface $overrides)`.
  - `ResolveRobotsDirectives implements RobotsDirectivesResolverInterface`: `__construct(SeoOverridesRepositoryInterface $overrides, SiteInfoProviderInterface $site)`.
- Overrides are looked up only when the context has both `objectType` and `objectId`.

**Rules the tests pin:**
- Canonical: `status !== 200` returns null. An override canonical wins. Otherwise the permalink. When `usesHttps()` the result has scheme `https`. No permalink returns null.
- Title: an override title is returned as is. Otherwise parts joined with `" {$separator} "`, empty parts skipped: content title, `sprintf($pageLabel, $page)` when `page >= 2` and kind is not `NotFound`, then site name. For `FrontPage` the parts are site name, page label, tagline.
- Description: override, else content summary; tags stripped, runs of whitespace collapsed to one space, trimmed; empty result is null. No truncation.
- Robots: site not public gives `noindex, nofollow`. Otherwise `Search` and `NotFound` give `noindex, follow`, and every other kind gives `max-image-preview:large`. Override directives are merged last and win.

- [ ] **Step 1: Write the failing tests**

```php
// helpers used below (define in tests/Pest.php)
// post(int $id = 7, int $page = 1, int $status = 200): PageContext  => new PageContext(PageKind::Singular, 'post', $id, $page, null, $status)

test('canonical forces https when the site uses https', function () {
    $links = new ArrayPermalinkSource();
    $links->set(post(), Url::fromString('http://example.com/hello/'));
    $r = new ResolveCanonicalUrl($links, new InMemorySeoOverridesRepository(), new ArraySiteInfoProvider(usesHttps: true));
    expect($r->resolve(post())->toString())->toBe('https://example.com/hello/');
});
test('canonical override wins, non-200 and unknown pages have none', function () {
    $links = new ArrayPermalinkSource();
    $links->set(post(), Url::fromString('https://example.com/hello/'));
    $overrides = new InMemorySeoOverridesRepository();
    $overrides->save('post', 7, new SeoOverrides(canonical: Url::fromString('https://example.com/original/')));
    $r = new ResolveCanonicalUrl($links, $overrides, new ArraySiteInfoProvider());
    expect($r->resolve(post())->toString())->toBe('https://example.com/original/')
        ->and($r->resolve(post(status: 404)))->toBeNull()
        ->and($r->resolve(post(id: 99)))->toBeNull();
});
test('title joins content title, page label and site name', function () {
    $content = new ArrayContentSource();
    $content->set(post(page: 2), 'Hello', null);
    $r = new ResolveTitle($content, new InMemorySeoOverridesRepository(), new ArraySiteInfoProvider(name: 'Site'));
    expect($r->resolve(post(page: 2)))->toBe('Hello - Page 2 - Site');
});
test('front page title is site name then tagline', function () {
    $r = new ResolveTitle(new ArrayContentSource(), new InMemorySeoOverridesRepository(), new ArraySiteInfoProvider('Site', 'Just a site'));
    expect($r->resolve(new PageContext(PageKind::FrontPage)))->toBe('Site - Just a site');
});
test('description strips tags, collapses whitespace and is not truncated', function () {
    $long = str_repeat('word ', 200);
    $content = new ArrayContentSource();
    $content->set(post(), 'T', "<p>Hello\n\n  <b>world</b></p> " . $long);
    $r = new ResolveDescription($content, new InMemorySeoOverridesRepository());
    expect($r->resolve(post()))->toStartWith('Hello world word')->and(strlen($r->resolve(post())))->toBeGreaterThan(900);
});
test('whitespace-only or tag-only summary gives no description', function (string $summary) {
    $content = new ArrayContentSource();
    $content->set(post(), 'T', $summary);
    expect((new ResolveDescription($content, new InMemorySeoOverridesRepository()))->resolve(post()))->toBeNull();
})->with(['', "  \n\t ", '<p> </p>', '<img src="a.png">']);
test('robots defaults by site visibility and page kind', function () {
    $public = new ResolveRobotsDirectives(new InMemorySeoOverridesRepository(), new ArraySiteInfoProvider(isPublic: true));
    $private = new ResolveRobotsDirectives(new InMemorySeoOverridesRepository(), new ArraySiteInfoProvider(isPublic: false));
    expect($public->resolve(post())->toString())->toBe('max-image-preview:large')
        ->and($public->resolve(new PageContext(PageKind::Search))->toString())->toBe('noindex, follow')
        ->and($private->resolve(post())->toString())->toBe('noindex, nofollow');
});
test('robots override merges last', function () {
    $overrides = new InMemorySeoOverridesRepository();
    $overrides->save('post', 7, new SeoOverrides(robots: RobotsDirectives::none()->with('noarchive')));
    $r = new ResolveRobotsDirectives($overrides, new ArraySiteInfoProvider());
    expect($r->resolve(post())->toString())->toBe('max-image-preview:large, noarchive');
});
```

`tests/ContractsTest.php` calls `resolverContract()` once per resolver with in-memory collaborators.

- [ ] **Step 2: Run `composer install && vendor/bin/pest`.** Expected: FAIL, classes not found.
- [ ] **Step 3: Implement the four use cases** to the rules above.
- [ ] **Step 4: Run `vendor/bin/pest`.** Expected: PASS.
- [ ] **Step 5: Commit** `feat(wp-seo-app): add canonical, title, description and robots resolvers`.

---

### Task 6: Head pipeline and `AppServiceProvider`

**Files:**
- Create in `wp-seo-app/src/`: `Head/HeadContributorRegistry.php`, `Head/HtmlHeadRenderer.php`, `Head/Contributors/{TitleContributor,DescriptionContributor,RobotsContributor,CanonicalContributor}.php`, `UseCases/BuildHeadDocument.php`, `AppServiceProvider.php`
- Test: `tests/Head/*Test.php`, `tests/AppServiceProviderTest.php`; extend `tests/ContractsTest.php` with `headContributorRegistryContract(fn () => new HeadContributorRegistry())`

**Interfaces:**
- Consumes: resolvers from Task 5 through their contracts.
- Produces:
  - Each contributor takes its one resolver contract in the constructor. `DescriptionContributor` sets `MetaTag(Name, 'description', …)` only when non-null; `RobotsContributor` sets `MetaTag(Name, 'robots', $directives->toString())` only when not empty; `CanonicalContributor` sets `LinkTag('canonical', $url->toString())` only when non-null; `TitleContributor` always sets the title.
  - `BuildHeadDocument implements HeadDocumentBuilderInterface`: `__construct(HeadContributorRegistryInterface $registry)`. Returns the same `HeadDocument` instance for a context equal to the previous call's, and rebuilds otherwise.
  - `HtmlHeadRenderer implements HeadRendererInterface`. Output formats, each ending `"\n"`: `<title>…</title>`, `<meta name="…" content="…" />` or `<meta property="…" content="…" />`, `<link rel="…" href="…" />` with extra attributes appended in order. Values are escaped with `htmlspecialchars($v, ENT_QUOTES | ENT_SUBSTITUTE | ENT_HTML5, 'UTF-8', false)` (no double encoding). `render()` emits title (when set), then metas, then links.
  - `AppServiceProvider::register(SeoServiceContainer $container): void` binds `CanonicalUrlResolverInterface`, `TitleResolverInterface`, `DescriptionResolverInterface`, `RobotsDirectivesResolverInterface`, `HeadContributorRegistryInterface`, `HeadDocumentBuilderInterface`, `HeadRendererInterface`, one `add()` each, with collaborators fetched from the container inside the factory. The registry factory adds contributors `title` (10), `description` (20), `robots` (30), `canonical` (40).

- [ ] **Step 1: Write the failing tests**

```php
// helper in tests/Pest.php: container(): SeoServiceContainer  => new container with MemoryServiceProvider then AppServiceProvider registered

test('renderer escapes once and does not double-encode entities', function () {
    $r = new HtmlHeadRenderer();
    expect($r->renderMeta(new MetaTag(MetaAttribute::Name, 'description', 'Tom &amp; "Jerry" &#8220;x&#8221; <b>')))
        ->toBe('<meta name="description" content="Tom &amp; &quot;Jerry&quot; &#8220;x&#8221; &lt;b&gt;" />' . "\n");
});
test('renderer uses the property attribute for property tags', function () {
    expect((new HtmlHeadRenderer())->renderMeta(new MetaTag(MetaAttribute::Property, 'og:title', 'T')))
        ->toBe('<meta property="og:title" content="T" />' . "\n");
});
test('builds title, description, robots and canonical for a post', function () {
    $c = container();
    $c->get(PageContextProviderInterface::class)->set(post());
    $c->get(ContentSourceInterface::class)->set(post(), 'Hello', 'A summary.');
    $c->get(PermalinkSourceInterface::class)->set(post(), Url::fromString('https://example.com/hello/'));
    $doc = $c->get(HeadDocumentBuilderInterface::class)->build(post());
    expect($doc->title())->toBe('Hello')
        ->and($doc->meta('description')->content)->toBe('A summary.')
        ->and($doc->meta('robots')->content)->toBe('max-image-preview:large')
        ->and($doc->link('canonical')->href)->toBe('https://example.com/hello/');
});
test('no description tag when there is no summary', function () {
    $doc = container()->get(HeadDocumentBuilderInterface::class)->build(post());
    expect($doc->meta('description'))->toBeNull();
});
test('equal contexts reuse one document, different contexts rebuild', function () {
    $b = container()->get(HeadDocumentBuilderInterface::class);
    expect($b->build(post()))->toBe($b->build(post()))->and($b->build(post(id: 8)))->not->toBe($b->build(post()));
});
test('replacing one resolver changes the built document and nothing else', function () {
    $c = container();
    $c->add(TitleResolverInterface::class, fn () => new class implements TitleResolverInterface {
        public function resolve(PageContext $context): string { return 'Replaced'; }
    });
    $doc = $c->get(HeadDocumentBuilderInterface::class)->build(post());
    expect($doc->title())->toBe('Replaced')->and($doc->meta('robots')->content)->toBe('max-image-preview:large');
});
test('a plugin can remove a contributor', function () {
    $c = container();
    $c->get(HeadContributorRegistryInterface::class)->remove('description');
    $c->get(ContentSourceInterface::class)->set(post(), 'Hello', 'A summary.');
    expect($c->get(HeadDocumentBuilderInterface::class)->build(post())->meta('description'))->toBeNull();
});
```

- [ ] **Step 2: Run `vendor/bin/pest`.** Expected: FAIL, classes not found.
- [ ] **Step 3: Implement** registry, renderer, contributors, builder and provider.
- [ ] **Step 4: Run `vendor/bin/pest`.** Expected: PASS.
- [ ] **Step 5: Commit** `feat(wp-seo-app): add head document pipeline and service provider`.

---

### Task 7: WordPress infrastructure

**Files:**
- Create: `packages/wp-seo/wp-seo-infra-wp/{composer.json,phpunit.xml,tests/Pest.php}` (requires domain only; require-dev app and contract-tests)
- Create: `src/Adapters/*.php`, `src/Resolvers/*.php`, `src/WordPressServiceProvider.php` (see File Structure)
- Create: `tests/Support/FakeWordPress.php`, `tests/Support/wordpress-functions.php` (loaded from `tests/Pest.php`)
- Test: `tests/Resolvers/*Test.php`, `tests/Adapters/*Test.php`, `tests/WordPressServiceProviderTest.php`

**Test seam:** these classes call WordPress functions directly. `tests/Support/wordpress-functions.php` defines global stubs backed by static state on `FakeWordPress` (`$options`, `$conditionals`, `$queryVars`, `$queriedObjectId`, `$posts`, `$filters`, `reset()`): `apply_filters`, `add_filter`, `do_action`, `get_option`, `get_bloginfo`, `get_locale`, `is_ssl`, the `is_*` conditionals used below, `get_queried_object_id`, `get_queried_object`, `get_query_var`, `get_post`, `get_post_status`, `get_permalink`, `add_query_arg`, `trailingslashit`, `user_trailingslashit`, `get_comments_pagenum_link`, `single_post_title`, `single_term_title`, `post_type_archive_title`, `get_search_query`, `get_the_date`, `__`, `_x`, `wp_strip_all_tags`. `tests/Pest.php` calls `FakeWordPress::reset()` in `beforeEach`.

**Interfaces (namespace `SpeedySpec\WP\Seo\Infra\WP`):**
- `Adapters\WordPressPageContextProvider`: kind from conditionals in this order: `is_404` NotFound, `is_search` Search, `is_front_page` FrontPage, `is_home` PostsIndex, `is_singular` Singular, `is_post_type_archive` PostTypeArchive, `is_category`/`is_tag`/`is_tax` TermArchive, `is_author` AuthorArchive, `is_date` DateArchive, else Other. `objectType` is `'post'` for Singular, `'term'` for TermArchive, `'user'` for AuthorArchive, else null. `objectId` is `get_queried_object_id()` or null when `0`. `page` is `max(paged, page, 1)` from query vars. `status` is 404 for NotFound, else 200.
- `Adapters\WordPressPermalinkSource`: non-Singular returns null. Singular reproduces today's `wp_get_canonical_url()` body up to but excluding the `get_canonical_url` filter (null when the post is missing or not `publish`; pagination and `cpage` handling unchanged), wrapped with `Url::tryFromString`.
- `Adapters\WordPressSiteInfoProvider`: `get_bloginfo('name', 'display')`, `get_bloginfo('description', 'display')`, `(bool) get_option('blog_public')`, `is_ssl()`.
- `Adapters\WordPressContentSource`: `titleFor` returns `single_post_title('', false)` for Singular and PostsIndex, `single_term_title('', false)` for TermArchive, else null. `summaryFor` for Singular returns the post excerpt, else the post content; null otherwise.
- `Adapters\WordPressEventDispatcher`: `filter()` calls `apply_filters($name, $value, ...$arguments)`; `dispatch()` calls `do_action($event::class, $event)`.
- `Adapters\WordPressResponseHeaderEmitter`: calls `header()` unless `headers_sent()`.
- `Adapters\NullSeoOverridesRepository`: `find` returns `SeoOverrides::none()`; `save` and `delete` do nothing. It exists because WordPress-backed overrides are deferred to phase 5 and infra packages may not require each other.
- `Resolvers\WordPressTitleResolver implements TitleResolverInterface`: the current body of `wp_get_document_title()` moved verbatim, filters included.
- `Resolvers\WordPressRobotsDirectivesResolver implements RobotsDirectivesResolverInterface`: `RobotsDirectives::fromArray(apply_filters('wp_robots', []))`.
- `Resolvers\WordPressCanonicalUrlResolver implements CanonicalUrlResolverInterface`: `__construct(CanonicalUrlResolverInterface $inner)`. For Singular with an `objectId`, passes the inner result (as string, or `false` when null) through `apply_filters('get_canonical_url', $url, get_post($objectId))` and returns `Url::tryFromString()` of the result. Other kinds return the inner result.
- `WordPressServiceProvider::register(SeoServiceContainer $container): void`: binds the seven data and output contracts to the adapters; rebinds `TitleResolverInterface` and `RobotsDirectivesResolverInterface`; wraps `CanonicalUrlResolverInterface` with `$container->extend()` (so it must be registered after an application-layer provider; `extend()` throws if none is bound).

- [ ] **Step 1: Write the failing tests**

```php
test('robots resolver applies the wp_robots filter to an empty array', function () {
    add_filter('wp_robots', fn (array $r) => $r + ['max-image-preview' => 'large']);
    add_filter('wp_robots', fn (array $r) => $r + ['noindex' => true, 'follow' => false]);
    expect((new WordPressRobotsDirectivesResolver())->resolve(post())->toString())->toBe('max-image-preview:large, noindex');
});
test('robots resolver survives a filter returning garbage', function (mixed $garbage, string $expected) {
    add_filter('wp_robots', fn () => $garbage);
    expect((new WordPressRobotsDirectivesResolver())->resolve(post())->toString())->toBe($expected);
})->with([['noindex', ''], [null, ''], [['NOINDEX' => true, 'bad key' => true], 'noindex']]);
test('canonical passes through get_canonical_url with the post', function () {
    FakeWordPress::$posts[7] = (object) ['ID' => 7, 'post_status' => 'publish'];
    add_filter('get_canonical_url', fn ($url, $post) => $url . '?p=' . $post->ID);
    $r = new WordPressCanonicalUrlResolver(fixedCanonical('https://example.com/hello/'));
    expect($r->resolve(post())->toString())->toBe('https://example.com/hello/?p=7');
});
test('canonical is dropped when the filter returns garbage', function (mixed $garbage) {
    add_filter('get_canonical_url', fn () => $garbage);
    expect((new WordPressCanonicalUrlResolver(fixedCanonical('https://example.com/hello/')))->resolve(post()))->toBeNull();
})->with([false, '', '/relative/path', null]);
test('title follows the WordPress algorithm and its filters', function () {
    FakeWordPress::$conditionals = ['is_search' => true];
    FakeWordPress::$queryVars = ['s' => 'cats'];
    FakeWordPress::$options['blogname'] = 'Site';
    add_filter('document_title_separator', fn () => '|');
    expect((new WordPressTitleResolver())->resolve(new PageContext(PageKind::Search)))
        ->toBe('Search Results for &#8220;cats&#8221; | Site');
});
test('pre_get_document_title short-circuits', function () {
    add_filter('pre_get_document_title', fn () => 'Forced');
    expect((new WordPressTitleResolver())->resolve(post()))->toBe('Forced');
});
test('permalink source returns nothing for unpublished or missing posts', function () {
    FakeWordPress::$posts[7] = (object) ['ID' => 7, 'post_status' => 'draft'];
    $s = new WordPressPermalinkSource();
    expect($s->permalinkFor(post()))->toBeNull()->and($s->permalinkFor(post(id: 99)))->toBeNull();
});
test('page context maps conditionals in priority order', function (array $conditionals, PageKind $kind) {
    FakeWordPress::$conditionals = $conditionals;
    expect((new WordPressPageContextProvider())->current()->kind)->toBe($kind);
})->with([
    [['is_404' => true, 'is_singular' => true], PageKind::NotFound],
    [['is_front_page' => true, 'is_home' => true], PageKind::FrontPage],
    [['is_home' => true], PageKind::PostsIndex],
    [['is_singular' => true], PageKind::Singular],
    [['is_tax' => true], PageKind::TermArchive],
    [[], PageKind::Other],
]);
test('provider wraps the app canonical resolver and rebinds title and robots', function () {
    $c = new SeoServiceContainer();
    (new AppServiceProvider())->register($c);
    (new WordPressServiceProvider())->register($c);
    expect($c->get(CanonicalUrlResolverInterface::class))->toBeInstanceOf(WordPressCanonicalUrlResolver::class)
        ->and($c->get(TitleResolverInterface::class))->toBeInstanceOf(WordPressTitleResolver::class)
        ->and($c->get(RobotsDirectivesResolverInterface::class))->toBeInstanceOf(WordPressRobotsDirectivesResolver::class)
        ->and($c->get(SeoOverridesRepositoryInterface::class))->toBeInstanceOf(NullSeoOverridesRepository::class);
});
```

`fixedCanonical(string $url)` is a `tests/Pest.php` helper returning an anonymous `CanonicalUrlResolverInterface`. Also call `eventDispatcherContract(fn () => new WordPressEventDispatcher())` and `resolverContract()` for the three resolvers.

- [ ] **Step 2: Run `composer install && vendor/bin/pest`.** Expected: FAIL, classes not found.
- [ ] **Step 3: Write `FakeWordPress` and the function stubs,** then implement adapters, resolvers and provider.
- [ ] **Step 4: Run `vendor/bin/pest`.** Expected: PASS.
- [ ] **Step 5: Commit** `feat(wp-seo-infra-wp): add WordPress adapters and filter-preserving resolvers`.

---

### Task 8: Legacy wrappers

**Files:**
- Create: `packages/wp-seo/wp-seo-legacy/{composer.json,phpunit.xml,tests/Pest.php,functions/seo.php,src/HeadAccess.php}`
  - `require`: `speedyspec/wp-seo-domain`, `speedyspec/wp-seo-app-implementation: ^1.0`, `speedyspec/wp-seo-infra-implementation: ^1.0`. `require-dev`: `speedyspec/wp-seo-app`, `speedyspec/wp-seo-infra-memory`, Pest. `autoload.files`: `functions/seo.php`.
- Modify: `packages/wp-theme/wp-theme-legacy/functions/general-template.php` (remove `wp_get_document_title`, `_wp_render_title_tag`), `functions/link-template.php` (remove `wp_get_canonical_url`, `rel_canonical`); delete `functions/robots-template.php` after moving its contents.
- Modify: `packages/wp-plugin/wp-plugin-legacy/functions/default-filters.php` — add `add_action( 'wp_head', 'wp_seo_render_head', 1 );` directly after the `wp_robots` line (currently line 353).
- Test: `tests/LegacyFunctionsTest.php`, `tests/Support/wordpress-functions.php` (stubs for `esc_attr`, `esc_url`, `current_theme_supports`, `get_option`, `is_embed`, `is_search`)

**Interfaces:**
- Consumes: only domain contracts, through `SeoServiceContainer::shared()`.
- Produces `SpeedySpec\WP\Seo\Legacy\HeadAccess`:
  - `static context(): ?PageContext` — null when `PageContextProviderInterface` is unbound.
  - `static document(): ?HeadDocument` — null when the provider or `HeadDocumentBuilderInterface` is unbound.
  - `static service(string $contract): ?object` — null when unbound.
- Produces in `functions/seo.php` (docblocks moved with the functions, plus `@since 0.1.0 speedyspec-wp-seo-legacy`):
  - `wp_get_document_title(): string` — the title resolver's result for the current context; `''` when unbound.
  - `_wp_render_title_tag(): void` — unchanged `current_theme_supports( 'title-tag' )` guard, then `echo '<title>' . $title . '</title>' . "\n"` with the document's title, unescaped as today; nothing when the document or title is missing.
  - `wp_robots(): void` — prints `"<meta name='robots' content='" . esc_attr( $content ) . "' />\n"` from the document's `robots` meta; nothing when absent.
  - `wp_robots_noindex`, `wp_robots_noindex_embeds`, `wp_robots_noindex_search`, `wp_robots_no_robots`, `wp_robots_sensitive_page`, `wp_robots_max_image_preview_large` — moved verbatim.
  - `wp_get_canonical_url( $post = null ): string|false` — builds a Singular `PageContext` (`null` uses the current context; an int is the id; an object uses `->ID`), carrying the current context's `page` only when the id matches the current object; returns the resolver's URL string or `false`.
  - `rel_canonical(): void` — nothing unless the current context is Singular with an object id; prints `'<link rel="canonical" href="' . esc_url( $url ) . '" />' . "\n"` from the document's `canonical` link.
  - `wp_seo_render_head(): void` — new. Echoes `renderMeta()` for every meta except key `robots` and `renderLink()` for every link except rel `canonical`, in document order. Never prints the title.

- [ ] **Step 1: Write the failing tests**

```php
// tests/Pest.php: beforeEach resets SeoServiceContainer::resetShared(); helper bindDefaults() registers
// MemoryServiceProvider then AppServiceProvider on SeoServiceContainer::shared() and returns it;
// helper output(callable $fn): string captures echo with ob_start.

test('prints nothing and does not fail when nothing is bound', function () {
    expect(wp_get_document_title())->toBe('')
        ->and(output('wp_robots') . output('rel_canonical') . output('_wp_render_title_tag') . output('wp_seo_render_head'))->toBe('')
        ->and(wp_get_canonical_url(7))->toBeFalse();
});
test('legacy output formats are byte-identical to WordPress', function () {
    $c = bindDefaults();
    $c->get(PageContextProviderInterface::class)->set(post());
    $c->get(ContentSourceInterface::class)->set(post(), 'Hello &#8220;x&#8221;', 'A summary.');
    $c->get(PermalinkSourceInterface::class)->set(post(), Url::fromString('https://example.com/hello/'));
    expect(output('_wp_render_title_tag'))->toBe("<title>Hello &#8220;x&#8221;</title>\n")
        ->and(output('wp_robots'))->toBe("<meta name='robots' content='max-image-preview:large' />\n")
        ->and(output('rel_canonical'))->toBe('<link rel="canonical" href="https://example.com/hello/" />' . "\n")
        ->and(wp_get_canonical_url(7))->toBe('https://example.com/hello/');
});
test('wp_seo_render_head prints only tags no legacy function prints', function () {
    $c = bindDefaults();
    $c->get(PageContextProviderInterface::class)->set(post());
    $c->get(ContentSourceInterface::class)->set(post(), 'Hello', 'A summary.');
    $c->get(PermalinkSourceInterface::class)->set(post(), Url::fromString('https://example.com/hello/'));
    expect(output('wp_seo_render_head'))->toBe('<meta name="description" content="A summary." />' . "\n");
});
test('rel_canonical stays silent off singular pages', function () {
    $c = bindDefaults();
    $c->get(PageContextProviderInterface::class)->set(new PageContext(PageKind::PostsIndex));
    expect(output('rel_canonical'))->toBe('');
});
test('title tag respects theme support', function () {
    bindDefaults();
    FakeTheme::$supportsTitleTag = false;
    expect(output('_wp_render_title_tag'))->toBe('');
});
test('legacy sources never name an implementation package', function () {
    $code = file_get_contents(__DIR__ . '/../functions/seo.php') . file_get_contents(__DIR__ . '/../src/HeadAccess.php');
    expect($code)->not->toContain('Seo\\App\\')->not->toContain('Seo\\Infra\\');
});
```

`FakeTheme::$supportsTitleTag` (default `true`, reset in `beforeEach`) backs the `current_theme_supports` stub.

- [ ] **Step 2: Run `composer install && vendor/bin/pest`.** Expected: FAIL, functions not defined.
- [ ] **Step 3: Implement `HeadAccess` and `functions/seo.php`,** moving the listed functions out of `wp-theme-legacy` and deleting `robots-template.php`.
- [ ] **Step 4: Add the `wp_seo_render_head` hook line** to `default-filters.php`.
- [ ] **Step 5: Verify the move is complete.** Run from the repository root:

```bash
grep -rn "^function \(wp_get_document_title\|_wp_render_title_tag\|wp_robots[a-z_]*\|wp_get_canonical_url\|rel_canonical\)\b" packages --include='*.php' | grep -v "wp-seo-legacy\|/vendor/"
```

Expected: no output.

- [ ] **Step 6: Run `vendor/bin/pest` in all six `wp-seo-*` packages.** Expected: PASS in each.
- [ ] **Step 7: Commit** `feat(wp-seo-legacy): route title, robots and canonical functions through the SEO contracts`.

---

## Out of scope for this plan

Structured data, Open Graph, robots.txt, sitemaps, hreflang, redirects, the URL-changed event, IndexNow, verification tags, WordPress-backed overrides, REST endpoints, the admin screen, canonical output on non-singular pages, any change to existing `X-Robots-Tag` `header()` calls, and wiring packages into a running site. Each later phase gets a design addendum first.
