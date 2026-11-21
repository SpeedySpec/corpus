# SpeedySpec Guide Hooks content Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** The Hooks seam, fully documented in both sections. Every Hook API function and global, `WP_Hook` with every method, and the hooks `{$hook_name}` get a WordPress page and a SpeedySpec page. Every function and method has tested examples. The seam gets its introduction, a porting guide with a checklist, and an architecture page. Coverage for the ported Hooks seam is then enforced.

**Architecture:** Mostly content (`content/…/*.mdx`, `seams/hooks/examples/…`), held to the guide's evidence rules by the toolchain. One small toolchain addition comes first: an evidence index that says which compat scenarios exercise which symbol, plus a rule that a ported seam's SpeedySpec page cites a scenario or break that actually exercises its symbol. One site fix: SpeedySpec class pages render their methods.

**Tech Stack:** As plans 1–3. MDX pages with `<Ev>`, `<Changed>` and the other components. PHP examples with `.expect.json`. `bun run guide …` stages and `bun run site:build`.

**Spec:** `docs/superpowers/specs/2026-11-20-speedyspec-guide-design.md` ("Evidence", "Examples", "Pages", "First seam"/"Order" step 2 and 3).

Plan 4 of 5. Plan 5 maps every other WordPress symbol into a seam, places the runtime's container and plugin-registry API in their own SpeedySpec seams, and records each cross-seam dependency's state.

## Global Constraints

- **Evidence.**
  - Every behaviour claim cites evidence: a compat scenario, a break, a SpeedySpec test, or WordPress 7.1.5 source lines.
  - SpeedySpec pages of Hooks (ported) cite scenarios, breaks or tests; source alone fails.
  - WordPress pages may cite `wp:` source. Each source citation carries the `hash` the validator prints.
  - Never cite PHPDoc as fact. Read the code.
- **WordPress is the ground truth for examples.** An example's `.expect.json` holds what WordPress 7.1.5 does.
  - When the runtime differs and a documented break explains it, the example runs on `wordpress` only, and the page carries a porting before/after pair.
  - When the runtime differs and no break explains it, **stop**. Record it in `docs/hooks-findings.md` (in the guide repo) as an undocumented incompatibility, for the server session. Leave the example on `wordpress` only, with the finding linked from the page. Never edit speedyspec-server.
- **Verdicts on SpeedySpec pages:**
  - `works-as-wordpress`: every scenario that exercises the symbol replays without a break.
  - `works-with-differences`: a documented break involves it. Each break becomes one `differences` entry with `break:`.
  - `breaks`: only if code using it cannot run at all.
- **Pages follow the conventions in plan 3**:
  - `content/wordpress/hooks/<slug>.mdx` and `content/speedyspec/hooks/<slug>.mdx`, with `symbol:` front matter
  - slugs as `site/src/model/slug.ts` makes them (`add_filter`, `class/WP_Hook`, `global/wp_filter`, `filter/{hook_name}`), with `/` written as a folder
  - examples in `seams/hooks/examples/<exampleFolder(id)>/<name>.php`, where `exampleFolder` is `class--WP_Hook--apply_filters` for a method
- **Writing style:** concise; one claim per sentence; headings `## Behaviour`, `## Override`, `## Porting` (SpeedySpec only) in that order when present. No marketing words.
- **Verification commands**, run from the guide repo with `GUIDE_FORK_DIR=.cache/fork-local/packages` (the fork commit isn't pushed):
  - `bun run guide examples`: exit 0, no failures
  - `bun run guide validate`: exit 0, no failures
  - `bun run site:build`: exit 0
  - `bun run test` and `bun run site:test`: green
- Commit locally. Pushing to `SpeedySpec/speedyspec-guide` is a separate step for Jacob to ask for.

## Review Focus

- A dynamic hook page (`filter/{hook_name}`) must not claim what any particular hook does. It says what `apply_filters()` passes through. Task 5 covers it.
- An example that depends on callback order at equal priority must cite the scenario pinning that order (`add_filter` ordering scenarios), not assume it. Task 3 covers it.
- `apply_filters_deprecated` and `do_action_deprecated` print a deprecation notice through `_deprecated_hook()`. Examples must fix what they assert (return value, not notice text), or define `WP_DEBUG` and assert the notice. Task 3 covers it.
- `WP_Hook`'s `ArrayAccess` and `Iterator` methods are used by WordPress core on `$wp_filter[$name]`. Their examples must show that use, not only direct calls. Task 5 covers it.
- An example whose expected output came from the runtime instead of WordPress would hide an incompatibility. Every `.expect.json` is first run on `wordpress` alone and must pass there before `runtime` is added to its sides. Every content task covers it.

---

## File Structure

```
packages/speedyspec-guide/
  packages/app/src/evidence-index.ts        (+ stage `evidence`, rule `CitesItsScenario`)
  site/src/pages/speedyspec/[seam]/[...slug].astro   (render methods on class pages)
  seams/hooks/seam.yaml                      (+ roles for runtime symbols, edges with citations, packages, crates)
  seams/hooks/examples/<folder>/<name>.php + .expect.json
  content/wordpress/hooks/{_seam.mdx, <slug>.mdx …}
  content/speedyspec/hooks/{_seam.mdx, _porting.mdx, _architecture.mdx, <slug>.mdx …}
  docs/hooks-findings.md                     (only if an undocumented incompatibility is found)
  guide.config.json                          ("requirePortedCoverage": true at the end)
```

---

### Task 1: Evidence index and the rule that a page cites its own scenarios

**Files:**
- Create: `packages/app/src/evidence-index.ts`, `packages/app/test/evidence-index.test.ts`
- Modify: `packages/contracts/src/index.ts` (`EvidenceSource.catalogue(area)`), `packages/adapter-compat-files-evidence/src/index.ts` and its test, `packages/app/src/validate/{rule,rules,run}.ts`, `packages/app/test/validate.test.ts`, `packages/app/src/pipeline.ts` (stage `evidence` → `generated/evidence-index.json`), `packages/cli/test/pipeline.test.ts` (stage list)

**Interfaces:**
- `EvidenceSource.catalogue(area: string): Promise<{ scenarios: { name: string; steps: { call: string }[] }[]; breaks: { scenario: string; step?: number; why: string }[] } | null>`.
- `buildEvidenceIndex(area: string, catalogue): EvidenceIndex` with `interface EvidenceIndex { [symbolId: string]: { scenarios: string[]; breaks: { scenario: string; step?: number; why: string }[] } }`. A step's `call` maps to an id:
  - `add_filter` → `function:add_filter`
  - `$h->apply_filters` and `$var->m` → `method:WP_Hook::apply_filters` (the hooks area's kept values are `WP_Hook` objects)
  - `WP_Hook::build_preinitialized_hooks` → `method:WP_Hook::build_preinitialized_hooks`
  - `new WP_Hook` → `class:WP_Hook`
  - `global:<op>` with first argument `wp_filter` → `global:wp_filter`

  A break is indexed under every symbol its scenario's steps call (or only that step's symbol when it names a `step`).
- Rule `CitesItsScenario`: on a SpeedySpec page (`symbol` set) of a ported seam, at least one cited scenario or break must be in the index entry for the page's symbol. For a class page, its methods' entries count too. It fails with "cites no scenario that exercises <id>; candidates: <first three>".

- [ ] **Step 1: Failing tests:**
  - `buildEvidenceIndex` over a three-scenario fixture maps each call form above to its id, and lists a stepped break under that step's symbol only.
  - `catalogue('hooks')` over the server fixture returns its scenarios and breaks; `catalogue('nope')` returns null.
  - `CitesItsScenario` fails a page citing an unrelated scenario, passes one citing a matching scenario or break, and passes a class page citing a method's scenario.
- [ ] **Step 2:** `bun run test`. Expected: FAIL. **Step 3:** Implement. **Step 4:** PASS, with `bun run typecheck`.
- [ ] **Step 5:** Real run: `bun run guide evidence`. Expected: `generated/evidence-index.json` has an entry for every Hooks function and method except those no scenario calls. List those in this task's commit message; they need SpeedySpec tests cited instead.
- [ ] **Step 6:** Commit `feat: evidence index, and a SpeedySpec page must cite its own scenarios`.

### Task 2: The Hooks seam map, and methods on SpeedySpec class pages

**Files:**
- Modify: `seams/hooks/seam.yaml`, `site/src/pages/speedyspec/[seam]/[...slug].astro`, `site/test/speedyspec.test.ts`

**Decisions:**
- **roles**, in this order:
  - `register`: `add_filter`, `add_action`
  - `run`: `apply_filters`, `apply_filters_ref_array`, `do_action`, `do_action_ref_array`, `apply_filters_deprecated`, `do_action_deprecated`
  - `remove`: `remove_filter`, `remove_all_filters`, `remove_action`, `remove_all_actions`
  - `inspect`: `has_filter`, `has_action`, `current_filter`, `current_action`, `doing_filter`, `doing_action`, `did_filter`, `did_action`
  - `classes-and-state`: `class:WP_Hook` and the four globals
  - `internals`: `_wp_call_all_hook`, `_wp_filter_build_unique_id`, `filter:{$hook_name}`, `action:{$hook_name}`
  - `speedyspec`: the runtime's `class:SpeedySpec\WpHooks\HookStorage`, `class:SpeedySpec\WpHooks\HookGlobal`, `class:SpeedySpec\WpHooks\FilterGlobal`, `class:SpeedySpec\WpHooks\CountGlobal` and `function:SpeedySpec\WpHooks\use_as_key`, `write_back_views`, `install_globals`, all listed in `symbols` too
  - `speedyspec-internals`: every `function:SpeedySpec\WpHooks\Native\…`, listed in `symbols`; their pages say "Used by SpeedySpec's PHP shims; not for plugins"
- **edges** (each with a `scenario` citation from the evidence index):
  - `add_filter` → `remove_filter`: `undone-by`
  - `add_action` → `remove_action`: `undone-by`
  - `add_action` → `add_filter`: `alias`
  - `has_action` → `has_filter`: `alias`
  - `remove_action` → `remove_filter`: `alias`
  - `remove_all_actions` → `remove_all_filters`: `alias`
  - `current_action` → `current_filter`: `alias`
  - `doing_action` → `doing_filter`: `alias`
  - `did_action` → `global:wp_actions`: `reads`
- **packages:** `[wp-plugin]`. **crates:** `[speedyspec-wp-hooks, speedyspec-wp-hooks-memory, speedyspec-wp-hooks-php, speedyspec-container, speedyspec-container-php, speedyspec-php-ext]`.
- The SpeedySpec symbol page renders `view.methods` as plan 3's WordPress class page does: a section per method with its anchor, signature, panel, examples and content. Then `pageOf` returns method anchors for SpeedySpec class pages that exist (remove the null case for methods of a class with a SpeedySpec page).

- [ ] **Step 1: Failing test:** in `speedyspec.test.ts`, a fixture SpeedySpec page for `class:WP_Hook` shows an `h2#apply_filters` and its signature, and `pageOf('method:WP_Hook::apply_filters', 'speedyspec', d)` is `/speedyspec/hooks/class/WP_Hook/#apply_filters` once that page exists (in `pages.test.ts`, replacing the null expectation).
- [ ] **Step 2:** `bun run site:test`. Expected: FAIL. **Step 3:** Implement the page section and the `pageOf` change; write the seam.yaml above. **Step 4:** PASS.
- [ ] **Step 5:** `bun run guide assign && bun run guide graph && bun run guide coverage && bun run guide validate`. Expected: every role id is in `hooks`, and the runtime ids are in `runtimeSeamOf` on the site. validate passes (edge citations exist and point to scenarios exercising both ends).
- [ ] **Step 6:** Commit `feat: the Hooks seam map, and methods on SpeedySpec class pages`.

### Tasks 3–7: the pages

Each of these tasks follows the same steps, for the symbols it lists:

- [ ] **Step 1: Examples first.** For each function or method, write its examples: at least one, and one per distinct use the code supports (for example `add_filter`: change a value, more than one argument, run last, a method callback). Each `.expect.json` starts with `"sides": ["wordpress"]`, and its `return` and `output` come from reading the 7.1.5 code.
- [ ] **Step 2:** `bun run guide examples`. Expected: every new example passes on `wordpress`. A failure means the expectation misread the code: re-read the code and correct the expectation, never the example's purpose.
- [ ] **Step 3:** Add `"runtime"` to each example's sides and run `bun run guide examples` again. Expected: PASS. A runtime failure follows the Global Constraints: either a documented break (move the example back to `wordpress` and add the porting pair), or a finding in `docs/hooks-findings.md`.
- [ ] **Step 4: Pages.**
  - **WordPress page** for each symbol: `## Behaviour` (each claim with `<Ev>`, scenarios where they exist, `wp:` source with hash otherwise) and `## Override` (the filters it fires, or that it can't be overridden).
  - **SpeedySpec page** for each symbol:
    - the `verdict` and its `differences`
    - `## Behaviour`, only where SpeedySpec differs or adds
    - `## Override`: how to swap behaviour on SpeedySpec, i.e. the hook store port, `HookStorage`, or "not overridable"
    - `## Porting`, when it has a break
- [ ] **Step 5:** `bun run guide validate` (citations, hashes, `CitesItsScenario`, `PortedSeamVerified`). Fix every failure it prints. Expected: exit 0.
- [ ] **Step 6:** `bun run site:build`. Expected: exit 0, and the pages render.
- [ ] **Step 7:** Commit `docs: <the task's title>`.

### Task 3: Registering and running hooks

Symbols: `add_filter`, `add_action`, `apply_filters`, `apply_filters_ref_array`, `do_action`, `do_action_ref_array`, `apply_filters_deprecated`, `do_action_deprecated`.

Must be covered, each with its scenario (and each confirmed by an example on WordPress before it is written down):
- `add_filter` always returns `true`.
- Callbacks at one priority run in the order added; lower priorities run first.
- `$accepted_args` limits the arguments passed.
- `add_action` is `add_filter`.
- `do_action` with no callbacks still counts in `did_action`.
- `do_action('x')` with no arguments passes one empty string to its callbacks.
- `_ref_array` variants pass their array as the argument list.

The deprecated variants fire `_deprecated_hook()` only when the hook has callbacks; examples assert the return value.

### Task 4: Removing and inspecting hooks

Symbols: `remove_filter`, `remove_all_filters`, `remove_action`, `remove_all_actions`, `has_filter`, `has_action`, `current_filter`, `current_action`, `doing_filter`, `doing_action`, `did_filter`, `did_action`.

Must be covered, each with its scenario:
- `remove_filter` needs the same priority the callback was added at, and returns whether it removed one.
- `remove_all_filters` with and without a priority.
- `has_filter` returns the priority, or `false`, or `true` without a callback.
- `current_filter` and `doing_filter` inside nested hooks.
- `did_filter` and `did_action` count calls, including with no callbacks.

### Task 5: `WP_Hook`, its methods, the internals and the dynamic hooks

Symbols:
- `class:WP_Hook`
- every `method:WP_Hook::…` (21)
- `_wp_call_all_hook`, `_wp_filter_build_unique_id`
- `filter:{$hook_name}`, `action:{$hook_name}`

Notes:
- The class page's examples use a `WP_Hook` directly (as `build_preinitialized_hooks` callers and core's `$wp_filter[$name]` do).
- `ArrayAccess` and `Iterator` method examples show core's use: `$wp_filter['init'][10]`, and iterating the callbacks.
- `_wp_filter_build_unique_id` examples cover the callback forms it ids (string, closure, `[object, method]`, `[class, method]`).
- The two dynamic-hook pages say what `apply_filters($hook_name, …)` and `do_action($hook_name, …)` pass through. They link to the functions, and make no claim about any particular hook.

### Task 6: The globals, the breaks and their porting pairs

Symbols: `global:wp_filter`, `global:wp_actions`, `global:wp_filters`, `global:wp_current_filter`.

Every documented break (`wp_filter_replace_wholesale`, `wp_hook_build_preinitialized_hooks`, `wp_filter_set_plain_array`, `wp_filters_read_and_write#5–7`) gets, on the SpeedySpec page of the global it concerns:
- a `differences` entry with its `break:`
- a porting before/after pair of examples: "before" runs on `wordpress` only, "after" on both
- a `## Porting` paragraph saying what to change

The WordPress pages describe each global's shape in 7.1.5 (`WP_Hook` objects keyed by name; counts keyed by name; the running stack) from source.

### Task 7: SpeedySpec's own Hooks API

Symbols: the `speedyspec` and `speedyspec-internals` roles of Task 2.

What each page covers:
- `HookStorage`: the PHP adapter of the hook store port.
- `HookGlobal`, `FilterGlobal` and `CountGlobal`: the proxies behind `$wp_filter`, `$wp_filters` and `$wp_actions`.
- `install_globals`, `write_back_views` and `use_as_key`.

Pages are `new-in-speedyspec`. They take their facts from the crates' code and tests, and cite `test:` paths (`speedyspec-wp-hooks-php::…`). The internals get one short page each: "Used by SpeedySpec's PHP shims; not for plugins", with the signature.

### Task 8: Seam introduction, porting guide, architecture

**Files:** `content/wordpress/hooks/_seam.mdx`, `content/speedyspec/hooks/{_seam,_porting,_architecture}.mdx`.

- **WordPress `_seam.mdx`:** what the seam is for, its four roles, and how the globals hold its state. Five to eight sentences.
- **SpeedySpec `_seam.mdx`:** that the seam is native, and what that means for plugins: the same functions, and a store in Rust behind proxies.
- **`_porting.mdx`:**
  - an overview
  - a **checklist**: one item per break pattern, each with a search pattern to run over plugin code (`\$wp_filter\s*=`, `\$GLOBALS\['wp_filter'\]\s*=`, `build_preinitialized_hooks`, `\$wp_filter\[[^\]]+\]\s*=\s*array`, `\$wp_filters\s*=`), what it means, and a link to the porting example
  - "What keeps working" with links
- **`_architecture.mdx`:**
  - the domain and the `HookStore` port (`speedyspec-wp-hooks`)
  - its adapters: memory (`speedyspec-wp-hooks-memory`) and PHP (`PhpHookStore`, through the `#[php_port]` twin)
  - the area that registers the functions and the global proxies (`speedyspec-wp-hooks-php`'s `WpHooksArea`)
  - the container binding (`HOOK_STORE`)
  - the patterns used and where: Ports and Adapters for the store, Proxy for the globals, Registry in the container
  - "How to change this": add a store adapter; which compat scenarios and tests must still pass; how to record new scenarios with `speedyspec-compat record hooks --wordpress 7.1.5`
  - each claim citing a `test:` or a scenario
  - a Mermaid sequence diagram of `apply_filters` from PHP into the store and back

Verification: Steps 5–7 of Tasks 3–7.

### Task 9: Enforce coverage, and the real build

- [ ] **Step 1:** Set `"requirePortedCoverage": true` in `guide.config.json`.
- [ ] **Step 2:** `bun run guide coverage && bun run guide validate`. Expected: exit 0, and `coverage.json` shows Hooks with no `missing`. A failure names the missing page; write it and rerun.
- [ ] **Step 3:** `bun run site:build`. Then check in the browser, light and dark: `/wordpress/hooks/`, `/wordpress/hooks/add_filter/`, `/wordpress/hooks/class/WP_Hook/`, `/speedyspec/hooks/`, `/speedyspec/hooks/global/wp_filter/`, `/speedyspec/porting/hooks/` and `/speedyspec/architecture/hooks/`. Look for: verdicts, examples marked as run on both sides where allowed, porting pairs, the checklist, and the architecture diagram.
- [ ] **Step 4:** Record in the README the Hooks page count, the example count and the number of findings (0 if none).
- [ ] **Step 5:** Commit `docs: the Hooks seam, fully documented; coverage enforced`.
