# Exploration: Micro-frontends for the detached admin and for plugin admin screens

**Status:** active
**Created:** 2026-11-21
**Author:** Claude, for the project owner
**Context:** Whether to cut the detached WordPress admin (`speedyspec-admin`) into micro-frontends, and how WordPress plugins could ship their own admin screens as micro-frontends.
**Approved by:** not recorded
**Socialized with:** not socialized

---

## Problem Statement

The detached admin replaces WordPress's `wp-admin` with a static site that a Node proxy serves on
its own origin. Its data comes from a PHP internal API, `wp-admin/v1`. Core screens are converted
in batches. Settings, Dashboard, Comments, Posts and Media are done, and about 72 screens remain.
Unconverted screens and plugin screens are shown classic, in a frame.

The owner asked what it would take to split this admin into micro-frontends, both for the admin
itself and so that plugins can ship admin screens of their own. The request called it "the Astro
admin". The admin started as an Astro plan, but Astro was dropped once runtime modules made it
redundant.

Any answer has to keep the project's rules:

- every dynamic PHP decision is computed by a `wp-admin/v1` endpoint and sent as data;
- writes through REST fire no `admin_init`, `wp_ajax_*` or `load-*.php`;
- contracts live apart from their implementations;
- each design pattern is named where it is used, and used only where it fits;
- no build step unless one is argued for explicitly.

## Current State

The admin is already composed at run time from independently served modules. In effect it is a
micro-frontend system, though nobody called it one.

- **Shell:** native ES modules, an import map, and a vendored full browser build of Vue 3.5 that
  includes the template compiler. There is no bundler.
- **Router manifest:** `GET wp-admin/v1/router` lists the pages the user may open, the menu,
  import-map entries contributed by WordPress and plugins, component replacements and slot
  modules. The client router resolves every admin URL against it, and polls a `version` stamp
  for live changes.
- **Contracts:** each stable piece (page, list table, inline edit, meta box, frame, notices) has a
  contract file that declares an id, a SemVer version, props, slots and emits, and a separate
  default implementation. The kit, `@wp-admin/kit`, also follows SemVer.
- **Extension:** PHP registers pages (`register_admin_page()` with a `module`), whole-component
  replacements (`register_admin_component_replacement()`, which declares the contract version
  implemented) and slot modules (`register_admin_slot_module()`). The browser keeps a replacement
  only when its major version matches the contract's. A replacement that fails to load, is
  incompatible or throws falls back to the default with a translated notice. Every slot entry is
  loaded and isolated on its own.
- **Gating:** the proxy serves admin-site files in three tiers. Public files are for sign-in.
  Shared files go to any signed-in user. Page files go only to users whose session lists that
  page's contract id.
- **Classic path:** plugin `add_menu_page()` callbacks and unconverted screens are shown as the
  real classic screen in a sandboxed iframe, with a frame channel back to the shell.
- **Untrusted HTML:** content HTML goes through an allowlist sanitiser over an inert parse.
  Captured plugin output is trusted, as classic WordPress trusts it.

The code shows gaps:

- Replacement and slot modules are put in every signed-in user's import map whatever their
  capability. Only page modules are filtered.
- Slot modules declare no contract version. A module written against version 1.x of a contract
  would break silently on 2.0.
- The kit exports a version, but nothing checks what a module needs.
- Plugin modules are WordPress files under `/wp-content`, and WordPress serves those to anyone.
- The shell sends only `frame-ancestors 'none'` as a Content Security Policy. The vendored
  compiler evaluates templates with `Function()`, so any script policy needs `'unsafe-eval'`.
- Plugin code runs in the shell's realm with nothing containing it. It can restyle the shell,
  patch globals or reach Vue's app context.

## Proposed Solution

Keep native ES-module federation through import maps as the backbone. Make it explicit and harden
it. Add an element tier for React and other frameworks behind the contract interface the shell
already has.

Owner answers (2026-11-21):

- **Trust.** Plugin JavaScript that shows up in the admin is trusted. There is no security
  boundary between plugins and the shell, so no sandbox tier. Isolation serves robustness only.
- **React.** It must be supported, so the element tier is required.
- **Exposure.** The proxy and WordPress both restrict plugin admin modules to admin users. The
  proxy serves a module only to a session whose manifest lists it. WordPress refuses direct
  requests for those files.
- **Compiled plugins.** Plugin developers ship precompiled render functions (compiled Vue, or JSX
  for React), never template strings, so the browser runs no compiler for them. The shell itself
  stays without a build step, so its own template strings, and with them `'unsafe-eval'`, remain.

### Architecture

The three seams do different jobs:

- **The screen is the micro-frontend unit.** It is a vertical slice: page contract,
  implementation directory, `wp-admin/v1` controller, parity checklist, end-to-end spec. It is
  gated by role and loaded lazily. Conversion batches already follow this seam, and plugin pages
  use it.
- **Parts are a shared platform, not micro-frontends.** The list table, inline edit, meta boxes,
  settings fields and kit stay in the shell's shared tier with SemVer contracts. Deploying them
  apart would add version skew and gain nothing.
- **Contracts are the extension seam.** Replacement and slot hooks work per contract id.

In Hexagonal terms:

- the shell runtime (router, loader, contract resolver, page host) is the core;
- the contracts and the `@wp-admin/api`, `@wp-admin/i18n` and `@wp-admin/kit` specifiers are its
  ports;
- core and plugin implementations are adapters;
- `wp-admin/v1` is the driven side that owns every decision.

There are two isolation tiers, and PHP chooses one per registration:

| Tier | Mount | Shared runtime | CSS | For |
| --- | --- | --- | --- | --- |
| `module` (default) | compiled Vue component | the shell's Vue | cascade layers, naming | core, and plugins written in Vue |
| `element` | custom element, or a `mount( el, ctx )` function | the shell's React, if the shell provides one | Shadow DOM with the admin sheets adopted | React and other plugins; pages and widgets |

The single component that resolves contracts builds the host for the tier: a Vue host or an
element host. Each host adapts its tier to the same contract.

### Data Model

The manifest's page entry and its slot and replacement entries gain these fields:

- `isolation`: `module` or `element`;
- `requires`: the router API version and a kit range;
- `implements`: on slot entries, the contract version written against;
- `capability`: on replacement and slot registrations, filtered in PHP;
- the module URLs each user may load, which the proxy keeps in its session cache to restrict
  module files;
- a `legacy` descriptor beside `module`, when a plugin registers both a native module and a
  classic callback.

To pause a plugin's admin modules, PHP stores a pause per user and per plugin, in the same way
WordPress's recovery mode stores paused extensions.

### Interfaces

- **PHP:** the three registration functions keep their signatures and accept these new arguments:
  `isolation`, `requires`, `capability`, `implements`, and a fallback to the classic callback.
  Page data still comes from `data_callback`. Action-specific nonces travel in that data.
- **Module:** the default export (a compiled Vue component), `contract`, `implementsContract`,
  and a new `requires` export that the shell checks before mounting. Element-tier modules export a
  custom element or a `mount( el, ctx )` function that returns an unmount function. The shell
  warns about, and may later refuse, a plugin component that carries a `template` string.
- **Module files:** the proxy serves a plugin module only to a session whose manifest lists it.
  WordPress refuses direct requests for it. How it refuses is open: a route that checks the
  capability and streams the file, or a web-server rule plus the proxy's secret header.
- **Data:** every call goes through the shell's single API client, which holds and renews the
  REST nonce. Plugin writes go to the plugin's own REST namespace and do not fire admin hooks.
- **Frame:** plugins get the frame channel (menu counts, document title) through an injected
  port. Classic plugin screens may get a small, versioned `postMessage` protocol that checks the
  origin and the source window.
- **Recovery:** `POST wp-admin/v1/client-errors` reports an error raised on the first mount after
  a plugin module loads. PHP decides whether to pause, and the router leaves paused modules out
  and says so in a notice that offers to resume them. A safe-mode flag, also decided in PHP,
  leaves out every plugin module.

## Design Decisions

| Decision | Pattern | Where |
| --- | --- | --- |
| Implementations are linked when the manifest is read, not at build time | Plugin (PoEAA) | Router manifest, module loader |
| Contracts sit in their own files, apart from their implementations | Separated Interface (PoEAA) | Contract files beside the component files |
| One registry of pages, replacements and slots in PHP | Registry (PoEAA) | The admin page registry |
| Coarse answers made for one screen | Remote Facade with Data Transfer Objects (PoEAA) | Each `wp-admin/v1` controller, the manifest's page entry |
| One client for all calls to WordPress | Gateway (PoEAA) | The shell's API client |
| A replacement stands in for the default implementation | Strategy (GoF) | Contract resolution |
| Order of resolution: replacement, default, classic screen, failure | Chain of Responsibility (GoF) | Contract resolution and the page host |
| A failed module renders nothing | Null Object (GoF) | The loader's module-failure component |
| A host built for each tier | Factory Method and Adapter (GoF) | The component that resolves contracts |
| Answers update the frame through one channel | Mediator (GoF) | The frame channel |

Other decisions:

- **No build step for the shell.** Plugin authors compile their own modules (an owner decision),
  but the shell and the way modules are combined stay free of a build. Module Federation is
  rejected because it would put a build into the shell. Import maps (including
  `scopes`, should a plugin ever need a different version of a dependency) give most of what
  sharing dependencies offers.
- **The single-spa library is not adopted.** It would bring a second routing authority beside the
  single router endpoint. Only its mount and unmount lifecycle shape is borrowed for the element
  tier.
- **Plugin styles are wrapped in a cascade layer below the shell's.** A plain `<link>` stylesheet
  outranks every layer, which is why the wrapping is needed.
- **Content Security Policy is measured before it is enforced.** First a report-only policy. Then
  the directives that cost nothing are enforced: `object-src`, `base-uri` and `form-action`.
  Script and connect sources wait until the classic-asset path shrinks after the cut-over.

## Implementation Phases

Rough working days for one engineer, tests included. Nothing starts before the testing batch now in
progress has landed.

### Phase 1: Harden the federation

Capability-filtered imports, slot `implements`, `requires` checks, cascade layers for plugin
stylesheets, restricted module files in the proxy and in WordPress, a module contract that
requires compiled render functions, a plugin author guide and an example plugin. 6 to 9 days.

### Phase 2: Admin recovery mode

Client error reports, pause storage in PHP, router filtering, safe mode and notices. 3 to 5 days.

### Phase 3: The element tier, with React

An element host, the lifecycle adapter, adopted admin stylesheets, how React is provided, and a
compiled React example plugin with end-to-end tests. 4 to 7 days.

### Phase 4: Fall back to the classic screen

Plugin pages registered both ways fall back to the classic screen when the native page fails, and
classic screens may get the optional message protocol. 3 to 5 days.

### Phase 5: Content Security Policy

Report-only first, then the directives that cost nothing. `'unsafe-eval'` stays while the shell
compiles its own templates. 1 to 2 days, then measuring.

In total, about 3.5 to 5.5 weeks.

## Fleet Impact

- **Core screens and conversion batches:** none. A converted screen already is the per-screen
  unit.
- **The cut-over:** none, though recovery mode and capability-filtered imports should land before
  it.
- **Plugins that use the classic path:** unchanged.
- **Plugins already registering replacements or slots:** they gain optional declarations. A slot
  entry with no `implements` keeps working, with a warning.
- **The existing unit and end-to-end suites:** stay valid while the contract-resolving component
  keeps its props and events.

Established by reading the code. Nothing was run.

## Open Questions

Answered on 2026-11-21: trust (trusted, no sandbox), React (supported), exposure (restricted by
the proxy and by WordPress) and compiling (plugins ship compiled render functions). Still open:

1. **Providing React.** Should the shell vendor one ES-module React and map `react`, `react-dom`
   and `@wordpress/element` as reserved specifiers, as it does Vue? Or should each plugin bundle
   its own? Sharing one copy is recommended: hooks break when two copies render one tree.
2. **Restricting files in WordPress.** Should WordPress serve module files through a route that
   checks the capability, or through a web-server rule plus the proxy's secret header? The route
   is recommended first.
3. **Core templates.** Keep compiling the shell's own templates in the browser, which keeps
   `'unsafe-eval'`, or precompile core too, which means a build step for the shell?
4. **Vue upgrades.** Should a new major version of Vue count as a new router API version?
5. **Core bundles.** Should groups of core screens ever deploy apart from each other, or stay in
   one site? This exploration recommends one site.
6. **Plugin namespace.** Plugin writes in the plugin's own REST namespace, or in an extension
   namespace under `wp-admin/v1`, which carries admin context on reads?
7. **Recovery.** Who may pause and resume a plugin's admin modules?
8. **Fallback.** Should a failing native plugin page fall back to its classic screen
   automatically?

## Spike Findings

No spikes were run. What reading the code established:

- A fixture plugin in the end-to-end suite already proves a plugin page, slots in the menu and in
  General Settings, and compatible, incompatible and throwing replacements.
- The loader refuses to remap the reserved specifiers (`vue`, `@wp-admin/*`), refuses URLs from
  other origins, and keeps the first URL for each specifier. A plugin update therefore needs a
  reload, which the live-update check already announces.
- Adding a second import map after modules have loaded depends on browser support. A plugin module
  that imports another plugin's bare specifier depends on it too.
- The vendored Vue build compiles templates with `Function()`.

## Success Criteria

- A plugin can ship a compiled native admin page, slot entry or replacement, in Vue or in React,
  and can declare the contract, kit and API versions it needs.
- No one but a signed-in admin user whose manifest lists a plugin module can fetch it, whether
  through the proxy or from WordPress directly.
- A plugin that fails to load, is incompatible, throws while rendering or breaks the page at import
  time never leaves a core screen unusable. The user can always reach safe mode.
- No user receives a plugin module their capabilities do not allow.
- Plugin CSS cannot override the shell's frame by accident.
- Classic plugin screens keep working in the frame until their authors convert them.

## Non-Goals

- Rebuilding hosted screens: the editors, Customizer, Connectors, Font Library, Patterns and the
  media grid.
- A build step for the shell. Plugin authors compile their modules; the shell does not.
- A security boundary between plugins and the shell.
- Running the admin as several independently deployed core sites.
- Changing the public `wp/v2` REST API.

## Alternatives Considered

### Option: Module Federation

It negotiates shared dependencies at run time, but every host and remote needs a bundler build.
Rejected because of the no-build rule.

### Option: The single-spa library

Its explicit mount lifecycle works with any framework, but it duplicates the router and the page
host and would be a second routing authority. Only the lifecycle shape is kept.

### Option: Custom elements as the default

They work with any framework, and Shadow DOM isolates CSS. But native slots cannot pass data, and
the core parts' slots must pass rows and columns. Classic scripts and id references also cannot
cross shadow roots. Kept as an opt-in tier.

### Option: Sandbox plugins in iframes

It isolates the most, but layout, focus, modals and history cost a great deal, and it gains no
security while the plugin's PHP stays unrestricted. Rejected: the owner treats plugin JavaScript
in the admin as trusted.

## Dependencies

- The testing batch now in progress, which should land before any of this starts.
- The detached-admin design: one router endpoint, contracts with replace and slot hooks,
  unauthorized by default.
- The cut-over, which waits for every core screen to be converted.

## References

- The detached-admin design and its later decisions for the Settings, Dashboard, Comments and
  Posts batches, in the WordPress fork's working documents.
- `speedyspec-admin`: the shell's runtime (loader, contract resolver, slots, page host, router,
  live-update check, API client, legacy frame), its contract files and kit, and its proxy and
  file tiers.
- The WordPress fork: the page registration functions and registry, the router controller's import
  building, the resolver for classic pages, and the detached-admin fixture plugin with its
  end-to-end specs.
