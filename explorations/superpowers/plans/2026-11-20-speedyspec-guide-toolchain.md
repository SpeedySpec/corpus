# SpeedySpec Guide toolchain Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** The build toolchain of `speedyspec-guide`: it extracts the WordPress 7.1.5 inventory and call graph, assigns symbols to seams, finds cross-seam dependencies and cycles (including through hooks), compares SpeedySpec's `packages/` fork with 7.1.5, computes WordPress "since" history, and validates citations, writing everything the site will render as `generated/*.json`.

**Architecture:** Hexagonal. A plain-object domain package, a contracts package of ports, one package per adapter, an app package of use cases, and a CLI that is the composition root. The build is Pipes and Filters (`extract → fork → assign → graph → history → validate`), each stage reading and writing files in `generated/`. Validation rules are Specifications, reported together.

**Tech Stack:** Bun 1.3 workspaces, TypeScript (strict), `bun test`; PHP 8.4 with `nikic/php-parser` ^5 for the extractor; `yaml` for seam files; git.

**Spec:** `docs/superpowers/specs/2026-11-20-speedyspec-guide-design.md`

This is plan 1 of 4. Later plans: 2. examples runner crate (`speedyspec-guide-examples`, Reflection inventory, SpeedySpec history); 3. Astro site (layouts, seam rail, relationship panel, Mermaid, Pagefind, `/wordpress/` and `/speedyspec/`); 4. Hooks content, the full WordPress seam map, and each SpeedySpec dependency's state (calls PHP, behind a port, answered in Rust).

## Global Constraints

- Repository: `packages/speedyspec-guide` (its own git repo, branch `main`). Creating the GitHub remote `SpeedySpec/speedyspec-guide` needs Jacob's yes first; until then commit locally only.
- WordPress documented: **7.1.5**, tarball `https://wordpress.org/wordpress-7.1.5.tar.gz`, sha256 `053857d980eaee8702c617c685948becb9be2523ba0dcbc98e4dbb2300367c79`.
- speedyspec-server pinned at `8fc498313cd9b6cd381fed1fc68e2e106a86b3b3` (`main`), repo `https://github.com/SpeedySpec/speedyspec-server.git`. Never edit speedyspec-server.
- SpeedySpec fork tree: `https://github.com/SpeedySpec/wordpress-develop.git`, path `packages/`, commit pinned in `sources.json`.
- WordPress release tags come from `https://github.com/WordPress/wordpress-develop.git`, read-only; only tags matching `^\d+(\.\d+)+$`.
- The extractor reads signatures and code only, never comments or `@since`.
- Domain objects are plain and know nothing about storage; ports live in `@speedyspec-guide/contracts`, apart from adapters.
- Generated files are never hand-edited, except `generated/wordpress-history.json`, which is committed as a cache.
- Symbol ids: `function:<name>`, `class:<Name>`, `method:<Class>::<name>`, `constant:<NAME>`, `global:<name>`, `filter:<hook>`, `action:<hook>`. A dynamic hook name keeps its template, e.g. `filter:pre_option_{$option}`.
- Do not commit `docs/superpowers` plans or specs, or `.superpowers/`.

## Review Focus

- A dynamic hook name with several variables (`{$meta_type}_{$option}`) or array access (`{$args['type']}`) must still become one pattern that matches literal names; the pattern test in Task 6 covers it.
- A `do_action` whose name is a concatenation with a function call (`'x_' . get_post_type()`) can't be turned into a template; it must be listed as unresolved, not dropped. Task 3 covers it.
- Closures and arrow functions passed to `add_filter` in WordPress's own code are callbacks without a name; they must not crash the graph and must not create edges. Task 3 covers it.
- A symbol defined twice under a `function_exists` guard (pluggable functions, `compat.php` polyfills) must be one symbol, flagged `pluggable: true`. Task 3 covers it.
- Old WordPress tags whose code has no `src/` folder (pre-3.7 layout) must still be parsed for history. Task 8 covers it.

---

## File Structure

```
packages/speedyspec-guide/
  package.json                    bun workspaces, scripts: test, typecheck, guide
  tsconfig.json  .gitignore  README.md
  sources.json                    pinned WordPress, server, fork
  guide.config.json               { "requireAllAssigned": false }
  seams/hooks/seam.yaml           first seam (members only; pages come in plan 4)
  generated/                      stage outputs (gitignored except wordpress-history.json)
  packages/
    domain/src/                   version.ts citation.ts symbol.ts seam.ts graph.ts verdict.ts index.ts
    contracts/src/index.ts        ports
    app/src/                      assign.ts graph.ts cycles.ts fork.ts history.ts validate/*.ts pipeline.ts
    adapter-wordpress-release/src/index.ts
    adapter-php-parser-source/    extractor/{composer.json,extract.php,src/*.php}  src/index.ts
    adapter-seam-files/src/index.ts
    adapter-compat-files-evidence/src/index.ts
    adapter-git-tag-history/src/index.ts
    cli/src/main.ts               composition root
  tests/fixtures/                 php/, compat/, seams/, git/
```

Each package's tests sit in `packages/<name>/test/*.test.ts`.

---

### Task 1: Repository, domain and contracts

**Files:**
- Create: `package.json`, `tsconfig.json`, `.gitignore`, `README.md`, `sources.json`, `guide.config.json`
- Create: `packages/domain/src/{version,citation,symbol,seam,graph,verdict,index}.ts`, `packages/domain/test/{version,citation}.test.ts`
- Create: `packages/contracts/src/index.ts`

**Interfaces:**
- Produces (domain):
  - `class Version { static parse(s: string): Version; compare(o: Version): -1|0|1; toString(): string }` — dotted numeric, any number of parts (`1.5.1.2`, `7.1.5`); `"unreleased"` parses to `Version.UNRELEASED`, which sorts after every release. Throws `InvalidVersion` otherwise.
  - `type Citation = { kind: 'scenario'; area: string; name: string } | { kind: 'break'; area: string; scenario: string; step?: number } | { kind: 'test'; path: string } | { kind: 'source'; base: 'wp' | 'ss'; file: string; from: number; to: number; hash?: string }` and `parseCitation(attrs: Record<string,string>): Citation`. Source form: `wp:wp-includes/plugin.php#L120-L134`; scenario/break attrs carry `area` (default `hooks`).
  - `type SymbolKind = 'function'|'class'|'method'|'constant'|'global'|'filter'|'action'`; `interface GuideSymbol { id: string; kind: SymbolKind; name: string; file: string; line: number; signature?: string; pluggable?: boolean; dynamic?: boolean; hash?: string }` (`hash` is the normalized syntax-tree hash).
  - `interface Seam { id: string; name: string; job: string; ported: boolean; files: string[]; symbols: string[]; exclude: string[]; roles: Record<string,string[]>; edges: Edge[] }`
  - `type EdgeKind = 'calls'|'reads'|'writes'|'fires'|'runs'|'matches'|'alias'|'undone-by'`; `interface Edge { from: string; to: string; kind: EdgeKind; at?: Citation }`; `interface Unresolved { from: string; at: Citation; code: string }`
  - `interface Inventory { base: 'wp'|'ss'; version: string; symbols: GuideSymbol[]; edges: Edge[]; unresolved: Unresolved[] }`
  - `type ForkVerdict = 'same'|'modified'|'removed'|'added'`
- Produces (contracts): `SymbolSource { extract(root: string, base: 'wp'|'ss', version: string): Promise<Inventory> }`, `ReleaseSource { fetch(version: string): Promise<string /* extracted root */> }`, `SeamRepository { all(): Promise<Seam[]> }`, `EvidenceSource { scenarioExists(area, name): Promise<boolean>; breakExists(area, scenario, step?): Promise<boolean>; testExists(path): Promise<boolean>; sourceLines(base, file, from, to): Promise<string|null> }`, `ReleaseHistory { tags(): Promise<string[]>; checkout(tag: string): Promise<string> }`.

- [ ] **Step 1:** `git init -b main packages/speedyspec-guide`; write `package.json` (`"workspaces": ["packages/*"]`, scripts `test: bun test`, `typecheck: tsc -b`, `guide: bun packages/cli/src/main.ts`), strict `tsconfig.json` with project references, `.gitignore` (`node_modules`, `.cache`, `generated/*`, `!generated/wordpress-history.json`), and `sources.json` with the Global Constraints' values (fork commit: `git -C <wordpress-develop> rev-parse origin/trunk`).
- [ ] **Step 2: Write failing tests** `version.test.ts`: `Version.parse('1.5.1.2').compare(Version.parse('1.5.1')) === 1`; `parse('7.1.5') < parse('7.10.0')`; `UNRELEASED` sorts after `parse('99.0')`; `parse('7.1-beta1')` throws `InvalidVersion`. `citation.test.ts`: `parseCitation({source:'wp:wp-includes/plugin.php#L120-L134'})` deep-equals `{kind:'source',base:'wp',file:'wp-includes/plugin.php',from:120,to:134}`; `#L7` gives `from:7,to:7`; `{scenario:'add_filter_returns_true'}` gives area `hooks`; `{source:'xx:a.php#L1'}` throws.
- [ ] **Step 3:** Run `bun test packages/domain` — FAIL (modules missing).
- [ ] **Step 4:** Implement the domain files and the contracts package with the Interfaces above.
- [ ] **Step 5:** Run `bun test packages/domain && bun run typecheck` — PASS.
- [ ] **Step 6:** Commit `feat: domain and contracts for the guide toolchain`.

### Task 2: WordPress release adapter

**Files:** Create `packages/adapter-wordpress-release/src/index.ts`, `test/release.test.ts`

**Interfaces:** Consumes `ReleaseSource`. Produces `class WordPressRelease implements ReleaseSource { constructor(sources: { url: string; sha256: string }, cacheDir: string, fetchImpl?: typeof fetch) }`; `fetch` returns `<cacheDir>/wordpress-<version>/wordpress`.

- [ ] **Step 1: Failing tests:** with a `fetchImpl` serving a tiny `.tar.gz` fixture built in the test: correct sha → returns a dir containing `wordpress/wp-includes/plugin.php`; a second call does not call `fetchImpl` (cached); wrong sha → rejects with message containing `sha256 mismatch` and leaves no directory behind.
- [ ] **Step 2:** Run — FAIL.
- [ ] **Step 3:** Implement: download to a temp file, verify sha256 (`Bun.CryptoHasher`), extract with `tar -xzf` into a temp dir, rename into place.
- [ ] **Step 4:** Run — PASS. Commit `feat: fetch and verify the pinned WordPress release`.

### Task 3: PHP extractor

**Files:** Create `packages/adapter-php-parser-source/extractor/{composer.json,extract.php}`, `extractor/src/{Collector.php,HookName.php,Hasher.php}`; fixtures `tests/fixtures/php/{plugin.php,class-hook.php,pluggable.php,dynamic.php,callbacks.php}`; `packages/adapter-php-parser-source/test/extract.test.ts`.

**Interfaces:** Produces the CLI `php extract.php <root> <base> <version>` printing one `Inventory` JSON (Task 1 shape) to stdout. File paths are relative to `<root>`. For `ss`, `<root>` is the fork's `packages/` dir.

Behaviour the tests pin:
- Symbols: every function, class/interface/trait, method (`method:Class::name`, signature includes visibility and `static`), `define()`/`const` constant, and every global named in a `global $x;` statement (`global:x`, file/line of first occurrence).
- Signature: printed from the AST with the standard pretty printer, parameters with types and defaults, return type; no body, no comments.
- `hash`: sha256 of the pretty-printed node with all comments and attributes stripped.
- A function declared inside `if ( ! function_exists( 'x' ) )` is one symbol with `pluggable: true` (the first declaration wins; a second identical guard does not duplicate it).
- Edges, each with a `source` citation of the call's line (`from = to = line`): `calls` for a direct call to a named function or `Class::method` / `new Class`; `reads`/`writes` for a global named in `global $x` (writes when it is assigned or passed by reference, reads otherwise) and for `$GLOBALS['x']`; `fires` from the enclosing symbol to `filter:<name>` for `apply_filters`, `apply_filters_ref_array`, `apply_filters_deprecated`, and to `action:<name>` for `do_action`, `do_action_ref_array`, `do_action_deprecated`; `runs` from `filter:<name>`/`action:<name>` to `function:<cb>` for `add_filter`/`add_action` with a literal hook name and a string callback, or `[ 'Class', 'm' ]` / `'Class::m'` → `method:Class::m`. Code at file top level uses `from: 'file:<path>'`.
- `HookName`: a string literal gives the name; an interpolated string or a concatenation of literals and variables (including `$a['k']` and `$o->p`) gives a template with each variable part written `{$...}` exactly as in source and marks the hook `dynamic: true`; anything else (a function call in the name) is an `Unresolved` entry with the printed expression as `code`.
- Hook symbols are created on first `fires` with `file`/`line` of that call.
- Closures, arrow functions and `[$this, 'm']` callbacks create no `runs` edge and no error. `call_user_func($var)`, `$obj->$m()`, `$fn()` are `Unresolved`.

- [ ] **Step 1:** `composer.json` requiring `nikic/php-parser: ^5.4`, PSR-4 `Guide\Extractor\` → `src/`; `composer install`.
- [ ] **Step 2: Failing tests** in `extract.test.ts`, running the CLI on `tests/fixtures/php` with `Bun.spawn` and asserting on the JSON: `function:add_filter` exists with signature `add_filter($hook_name, $callback, $priority = 10, $accepted_args = 1)`; `method:WP_Hook::apply_filters` exists; `global:wp_filter` exists and `function:add_filter` has a `writes` edge to it; `function:wp_mail` (fixture `pluggable.php`, guarded twice) appears once with `pluggable: true`; `function:update_option` fires `filter:pre_update_option_{$option}` (dynamic) and `action:updated_option`; `do_action( 'x_' . get_post_type() )` yields one `unresolved` entry whose `code` is `'x_' . get_post_type()`; `add_filter( 'the_content', 'wpautop' )` at top level yields edge `{from:'filter:the_content', to:'function:wpautop', kind:'runs'}`; `add_filter( 'a', function () {} )` yields no `runs` edge; the same function with different comments in two fixture copies has equal `hash`.
- [ ] **Step 3:** Run `bun test packages/adapter-php-parser-source` — FAIL.
- [ ] **Step 4:** Implement `Collector` (a `NodeVisitorAbstract` tracking the enclosing symbol), `HookName::from(Expr): array{name?:string, dynamic:bool, unresolved?:string}`, `Hasher::hash(Node): string`.
- [ ] **Step 5:** Run — PASS. Commit `feat: PHP extractor for symbols, calls, globals and hooks`.

### Task 4: Symbol source adapter

**Files:** Create `packages/adapter-php-parser-source/src/index.ts`, `test/source.test.ts`

**Interfaces:** Produces `class PhpParserSource implements SymbolSource { constructor(php = 'php') }`.

- [ ] **Step 1: Failing test:** `new PhpParserSource().extract('tests/fixtures/php','wp','7.1.5')` resolves to an `Inventory` with `base: 'wp'`, `version: '7.1.5'`, and every edge's `at` a `Citation` of kind `source`. A missing root rejects with a message naming the root and the extractor's stderr.
- [ ] **Step 2:** Run — FAIL. **Step 3:** Implement (spawn, parse, map). **Step 4:** Run — PASS.
- [ ] **Step 5:** Commit `feat: SymbolSource over the PHP extractor`.

### Task 5: Seams and assignment

**Files:** Create `packages/adapter-seam-files/src/index.ts`, `packages/app/src/assign.ts`, `seams/hooks/seam.yaml`, fixtures `tests/fixtures/seams/`, tests `packages/adapter-seam-files/test/seams.test.ts`, `packages/app/test/assign.test.ts`.

**Interfaces:**
- Produces `class SeamFiles implements SeamRepository { constructor(dir: string) }` reading `<dir>/*/seam.yaml`: `{ name, job, ported, files: [globs], symbols: [ids], exclude: [ids], roles: { <role>: [ids] }, edges: [{ from, to, kind, at: {<citation attrs>} }] }`; the folder name is the seam id.
- Produces `assign(inv: Inventory, seams: Seam[]): Assignment` with `interface Assignment { seamOf: Record<string,string>; unassigned: string[]; conflicts: { symbol: string; seams: string[] }[] }`.

Rules: a symbol listed in a seam's `symbols` belongs there; otherwise a symbol whose `file` matches a seam's `files` glob belongs there unless that seam `exclude`s it; a method belongs to its class's seam; a hook belongs to the seam of the symbols that fire it when they are all in one seam; a hook fired from more than one seam must be listed in some seam's `symbols`, else it is a conflict (this replaces the spec's "first in load order", which the extractor cannot see; noted for Jacob). Two seams claiming one symbol is a conflict.

`seams/hooks/seam.yaml`: `ported: true`, `files: [wp-includes/plugin.php, wp-includes/class-wp-hook.php]`, `exclude` the path helpers `function:plugin_basename`, `function:wp_register_plugin_realpath`, `function:plugin_dir_path`, `function:plugin_dir_url`, `function:register_activation_hook`, `function:register_deactivation_hook`, `function:register_uninstall_hook`; `symbols: [global:wp_filter, global:wp_actions, global:wp_filters, global:wp_current_filter]`; roles `register: [function:add_filter, function:add_action]`, `run: [function:apply_filters, function:apply_filters_ref_array, function:do_action, function:do_action_ref_array, function:apply_filters_deprecated, function:do_action_deprecated]`, `remove: [function:remove_filter, function:remove_all_filters, function:remove_action, function:remove_all_actions]`, `inspect: [function:has_filter, function:has_action, function:current_filter, function:current_action, function:doing_filter, function:doing_action, function:did_filter, function:did_action]`, `classes-and-state: [class:WP_Hook, global:wp_filter, global:wp_actions, global:wp_filters, global:wp_current_filter]`; `edges: []` (written with content in plan 4).

- [ ] **Step 1: Failing tests:** fixture seams `a` (files `a/*.php`) and `b` (symbols `[function:shared]`): a symbol in `a/x.php` maps to `a`; `function:shared` defined in `a/` maps to `b` (explicit wins); `method:A::m` follows `class:A`; a hook fired only from `a` maps to `a`; a hook fired from `a` and `b` and listed nowhere is in `conflicts`; a symbol matching no seam is in `unassigned`; a `seam.yaml` missing `job` makes `SeamFiles.all()` reject naming the file and the field.
- [ ] **Step 2:** Run — FAIL. **Step 3:** Implement (`Bun.Glob` for globs, `yaml` for parsing). **Step 4:** Run — PASS.
- [ ] **Step 5:** Commit `feat: seam files and symbol assignment`.

### Task 6: Dependency graph and cycles

**Files:** Create `packages/app/src/{graph,cycles,hook-pattern}.ts`, tests `packages/app/test/{graph,cycles,hook-pattern}.test.ts`

**Interfaces:**
- `hookPattern(template: string): RegExp` — each `{$...}` becomes `.+`, everything else is escaped; anchored.
- `buildGraph(inv: Inventory, a: Assignment): Graph` with `interface Graph { nodes: string[]; edges: Edge[]; crossSeam: { from: string; to: string; fromSeam: string; toSeam: string; kind: EdgeKind; at?: Citation }[] }`. Adds a `matches` edge from each dynamic hook to every literal hook of the same kind its pattern matches. `crossSeam` lists every `calls`/`reads`/`writes`/`fires`/`runs` edge whose ends are in different seams.
- `findCycles(g: Graph, a: Assignment): Cycles` with `interface Cycles { symbol: string[][]; seam: string[][] }`. Each cycle is a closed path (first element repeated last), found with Tarjan's strongly connected components, then one shortest cycle per component through its smallest node id (BFS); a self-loop is a cycle of length 1. The seam graph has an edge `S → T` for each `crossSeam` entry.

- [ ] **Step 1: Failing tests:** `hookPattern('{$meta_type}_{$option}').test('post_title')` true, `.test('_title')` false; `hookPattern('a.b_{$x}').test('aXb_1')` false (dot escaped). Graph: dynamic `filter:update_option_{$option}` gets a `matches` edge to literal `filter:update_option_blogname`. Cycles: `f calls g, g calls f` → `[['function:f','function:g','function:f']]`; through hooks, `f fires filter:x`, `filter:x runs g`, `g calls f` → one symbol cycle of those three nodes; a callback firing its own hook (`filter:x runs h`, `h fires filter:x`) → a cycle; seams `a→b→a` → `seam: [['a','b','a']]`; an acyclic graph → both lists empty.
- [ ] **Step 2:** Run — FAIL. **Step 3:** Implement. **Step 4:** Run — PASS.
- [ ] **Step 5:** Commit `feat: dependency graph with hooks, and cycle detection`.

### Task 7: Fork comparison

**Files:** Create `packages/app/src/fork.ts`, test `packages/app/test/fork.test.ts`

**Interfaces:** `compareFork(wp: Inventory, ss: Inventory): Record<string, ForkVerdict>` over function, class and method ids: `same` when both have it with equal `hash`, `modified` when hashes differ, `removed` when only in `wp`, `added` when only in `ss`. Constants, globals and hooks are not compared.

- [ ] **Step 1: Failing test** with two hand-built inventories covering all four verdicts and a global that is absent from the result.
- [ ] **Step 2:** FAIL. **Step 3:** Implement. **Step 4:** PASS. **Step 5:** Commit `feat: compare the packages fork with WordPress`.

### Task 8: WordPress history

**Files:** Create `packages/adapter-git-tag-history/src/index.ts`, `packages/app/src/history.ts`, tests `packages/adapter-git-tag-history/test/tags.test.ts`, `packages/app/test/history.test.ts`, fixture builder `tests/fixtures/git/make-repo.ts`.

**Interfaces:**
- `class GitTagHistory implements ReleaseHistory { constructor(remote: string, cacheDir: string) }`: `tags()` lists release tags (regex in Global Constraints) sorted by `Version`; `checkout(tag)` returns a directory with that tag's PHP: `src/` if the tag has one, else the repository root (old layout). Uses one blobless clone in `<cacheDir>/wordpress-develop.git` and `git archive <tag>` into `<cacheDir>/tags/<tag>`, removed after use by the caller.
- `computeHistory(history: ReleaseHistory, source: SymbolSource, cache: HistoryFile): Promise<HistoryFile>` with `interface HistoryFile { tags: string[]; symbols: Record<string, { since: string; removed?: string }> }`. Parses only tags not in `cache.tags`, in version order. `since` is the first tag containing the id; `removed` the first later tag without it (cleared if it comes back). Hooks count by id, so a literal hook's `since` is when its name first appears.

- [ ] **Step 1: Failing tests:** `make-repo.ts` builds a temp git repo with tags `1.0` (root layout, `a.php` defining `a()`), `3.7.0` (`src/a.php` with `a()` and `b()`), `3.7.1` (`src/b.php` only), and a tag `3.8-beta1`. `tags()` returns `['1.0','3.7.0','3.7.1']`. `computeHistory` from an empty cache gives `a: {since:'1.0', removed:'3.7.1'}`, `b: {since:'3.7.0'}`; run again with that result as cache and a `SymbolSource` spy: `extract` is not called.
- [ ] **Step 2:** FAIL. **Step 3:** Implement. **Step 4:** PASS.
- [ ] **Step 5:** Commit `feat: WordPress since and removed from release tags`.

### Task 9: Evidence and validation rules

**Files:** Create `packages/adapter-compat-files-evidence/src/index.ts`, `packages/app/src/validate/{rule,rules,citations-in-content,run}.ts`; fixtures `tests/fixtures/compat/hooks/{scenarios,breaks}.json`, `tests/fixtures/content/`; tests `packages/adapter-compat-files-evidence/test/evidence.test.ts`, `packages/app/test/validate.test.ts`.

**Interfaces:**
- `class CompatFilesEvidence implements EvidenceSource { constructor(serverDir: string, wordpressDir: string, forkDir: string) }`: scenarios and breaks read from `<serverDir>/compat/<area>/`; `testExists(path)` checks `<crate>::<test file stem>::<fn>` as `fn <name>` in `<serverDir>/crates/<crate>/tests/<stem>.rs`; `sourceLines('wp', …)` reads from `wordpressDir`, `'ss'` from `forkDir`.
- `citationsIn(contentDir: string): Promise<{ page: string; seam: string; section: 'wordpress'|'speedyspec'; citation: Citation }[]>` — scans `content/<section>/<seam>/**/*.mdx` for `<Ev …/>` and parses attributes.
- `interface Rule { id: string; check(ctx: ValidationContext): Promise<Failure[]> }`, `interface Failure { rule: string; where: string; message: string }`, `interface ValidationContext { inventory: Inventory; assignment: Assignment; seams: Seam[]; citations: Awaited<ReturnType<typeof citationsIn>>; evidence: EvidenceSource; history: HistoryFile; versionsOnPages: { page: string; version: string; base: 'wp'|'ss' }[]; config: { requireAllAssigned: boolean } }`.
- Rules: `EverySymbolInOneSeam` (conflicts always fail; unassigned fail only when `requireAllAssigned`), `CitationExists`, `CitedLinesUnchanged` (a source citation with `hash` fails when the sha256 of the current lines differs; without `hash` it fails with a message giving the hash to add), `PortedSeamVerified` (on `speedyspec` pages of a `ported` seam, a page whose only citations are `source` fails), `WordPressPagesCiteWordPress` (a `wordpress` page citing `ss:` fails), `VersionHasTag` (`wp` versions must be in `history.tags`; `ss` versions are checked in plan 2). `PortedSeamCovered` and `ExampleHasRun` come with the site and runner plans.
- `runRules(rules: Rule[], ctx): Promise<Failure[]>` runs every rule and returns all failures.

- [ ] **Step 1: Failing tests:** evidence: `scenarioExists('hooks','add_filter_returns_true')` true for the fixture, `'nope'` false; `breakExists('hooks','wp_filters_read_and_write',5)` true. Validation, each with a minimal context: a conflict yields an `EverySymbolInOneSeam` failure naming the symbol; an `<Ev scenario="nope"/>` yields `CitationExists`; a source citation whose stored hash no longer matches yields `CitedLinesUnchanged`; a `speedyspec/hooks` page citing only `wp:` source yields `PortedSeamVerified`; a `wordpress` page with `ss:` yields `WordPressPagesCiteWordPress`; a page naming WordPress `9.9.9` yields `VersionHasTag`; a context with three different faults returns three failures from one `runRules` call.
- [ ] **Step 2:** FAIL. **Step 3:** Implement. **Step 4:** PASS.
- [ ] **Step 5:** Commit `feat: evidence source and validation rules`.

### Task 10: Pipeline CLI

**Files:** Create `packages/app/src/pipeline.ts`, `packages/cli/src/main.ts`, test `packages/cli/test/pipeline.test.ts`; modify `README.md` (how to run each stage, env vars).

**Interfaces:**
- Stages, each reading the previous stages' files: `extract` → `generated/inventory-wp.json`, `generated/inventory-ss.json`; `fork` → `generated/fork.json`; `assign` → `generated/assignment.json`; `graph` → `generated/graph.json`, `generated/cycles.json`; `history` → `generated/wordpress-history.json`; `validate` → `generated/validation.json`, exit code 1 on any failure, every failure printed as `<rule> <where>: <message>`. `all` runs them in order and stops at the first stage that fails.
- `bun run guide <stage> [--offline]`. Inputs: `sources.json`; checkouts under `.cache/` (server at the pinned commit, fork at its commit via `git archive … packages`), overridable by `GUIDE_SERVER_DIR`, `GUIDE_FORK_DIR`, `GUIDE_WORDPRESS_DIR` for local runs. `--offline` skips `history` and uses the committed cache.

- [ ] **Step 1: Failing test:** with `GUIDE_*_DIR` pointing at the fixtures and `--offline`, `bun run guide all` in a temp copy writes every file listed above; `graph.json` has a `crossSeam` entry; `validation.json` lists the fixture's deliberate bad citation and the process exits 1.
- [ ] **Step 2:** FAIL. **Step 3:** Implement the composition root wiring the adapters. **Step 4:** PASS.
- [ ] **Step 5: Real run:** `bun run guide extract && bun run guide assign && bun run guide graph` against WordPress 7.1.5. Expected: `inventory-wp.json` has `function:add_filter` and `class:WP_Hook`; `assignment.json` maps all 25 ids in `seams/hooks/seam.yaml` roles to `hooks`; record the counts of symbols, unresolved entries and symbol cycles in the README.
- [ ] **Step 6:** Commit `feat: guide pipeline CLI`.

### Task 11: CI

**Files:** Create `.github/workflows/ci.yml`, `.github/workflows/history.yml`

- [ ] **Step 1:** `ci.yml` on pull request and push to `main`: set up Bun 1.3 and PHP 8.4 with Composer, cache `.cache/wordpress-7.1.5` keyed by the sha256 and `.cache/speedyspec-server` by the commit, then `bun install`, `bun run typecheck`, `bun test`, `bun run guide all --offline`.
- [ ] **Step 2:** `history.yml` weekly and on `workflow_dispatch`: runs `bun run guide history`, and opens a pull request when `generated/wordpress-history.json` changed.
- [ ] **Step 3:** Validate both with `actionlint` if available (else `bunx yaml-lint`). Expected: no errors.
- [ ] **Step 4:** Commit `ci: test, build and history workflows`. Do not push until the remote exists with Jacob's yes.
