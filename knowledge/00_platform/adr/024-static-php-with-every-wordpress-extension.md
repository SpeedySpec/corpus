# ADR-024: Static PHP 8.5 with every extension WordPress uses

**Status:** Accepted
**Date:** 2026-11-18
**Deciders:** both: the project owner ("the embed should include all extensions that WordPress requires"); the designer (the list)
**Exploration:** [PHP foundation](../speedyspec-server/explorations/2026-11-18-php-foundation.md)
**Related:** [ADR-016](016-rust-runs-as-an-embedded-php-extension.md), [ADR-023](023-shared-crates-in-the-server-workspace.md), [ADR-025](025-supervised-php-worker-pools.md)

---

## Context

PHP is static and embedded ([ADR-016](016-rust-runs-as-an-embedded-php-extension.md)), so the host needs no PHP of its
own. That also means the host cannot add an extension later. WordPress, its Site Health screen and its plugins expect
a wide set of extensions, some of them hard to link statically, such as `imagick`, `intl` and `gd` with AVIF.

### Why this matters

A missing extension shows up late: as a Site Health warning, a broken image upload, or a plugin fatal error on a live
site. If hard extensions were dropped quietly to make the build pass, nobody would notice until then.

---

## Decision

The embedded PHP is static PHP 8.5, not thread-safe, built with static-php-cli from a pinned recipe, and it includes
every extension WordPress's Site Health checks for plus what WordPress otherwise relies on.

### Rules

1. PHP 8.5, non-thread-safe, built with static-php-cli from the pinned recipe (`php-version`, `scripts/build-php.sh`),
   with the `embed` SAPI.
2. Targets: macOS arm64, and Linux musl on x86_64 and aarch64.
3. **Site Health, required:** `hash`, `json`, `pcre`.
4. **Site Health, recommended:** `curl`, `dom`, `exif`, `fileinfo`, `imagick`, `mbstring`, `mysqli` with `mysqlnd`,
   `sodium`, `openssl`, `libxml`/`xml`, `xmlreader`, `simplexml`, `zip`, `filter`, `gd` (JPEG, PNG, WebP, AVIF,
   FreeType), `iconv`, `intl`, `zlib`.
5. **What WordPress otherwise relies on:** `opcache`, `ctype`, `tokenizer`, `xmlwriter`, `bcmath`, `session`, `pdo` with
   `pdo_mysql`, `pdo_sqlite` and `sqlite3`, and `phar`.
6. `mcrypt` is not included; PHP removed it in 7.2.
7. The list is a hard requirement. An extension that will not build stops the work and goes back to the project
   owner. It is never dropped quietly.

### What this looks like

This is a build decision, so there is no code shape. The recipe's list is the decision in config:

```sh
# scripts/build-php.sh (php-version pins 8.5.12)
extensions="bcmath,ctype,curl,dom,exif,fileinfo,filter,gd,iconv,imagick,intl,mbstring,mysqli,mysqlnd,opcache,openssl,pdo,pdo_mysql,pdo_sqlite,phar,session,simplexml,sodium,sqlite3,tokenizer,xml,xmlreader,xmlwriter,zip,zlib"
```

### Patterns

None. This is a build decision.

---

## Consequences

### Positive

1. WordPress, Site Health and common plugins find every extension they expect.
2. The host needs no PHP and no extension packages.
3. The server and the CLI run WordPress code with the same extensions.

### Negative

1. Static builds of `imagick`, `intl` and `gd` with AVIF are slow and fragile.
   **Mitigation:** CI caches the build, keyed on `php-version` and `scripts/build-php.sh`.
2. Linux x86_64 was not covered at first: `intl` crashed under emulation.
   **Mitigation:** traced to an export-dynamic relocation bug and fixed in SpeedySpec/speedyspec-server#9; CI now has
   a Linux x86_64 PHP job.
3. Adding an extension means rebuilding and releasing the runtime.
   **Mitigation:** the runtime releases on its own, without a server release
   ([ADR-025](025-supervised-php-worker-pools.md)).

### Trade-offs

**We prioritise:**
- Every extension WordPress uses, on every target.
- A host with no PHP.

**We accept:**
- Long, fragile static builds.
- A runtime release for any extension change.

---

## Fleet impact

- **speedyspec-php-runtime:** is built from this PHP, on every target.
- **speedyspec-server:** holds the recipe; the server binary itself links no PHP.
- **speedyspec-cli:** its PHP build uses the same extension list
  ([ADR-023](023-shared-crates-in-the-server-workspace.md)). The CLI's `scripts/build-php.sh` lists the same
  extensions as of 2026-11-21.
- **PHP apps, plugins and themes:** may rely on any listed extension, and on no other.

---

## Enforcement

### Automated

- `cargo test -p speedyspec-php --test extensions` boots the embedded PHP, checks every listed extension with
  `extension_loaded`, probes key functions and classes, and checks `gd`'s JPEG, PNG, WebP, AVIF and FreeType support.
  It prints the missing names and exits 1.
- It runs in the server's CI `php` job, in the "Extensions" step, on macOS arm64, Linux x86_64 and Linux aarch64. It
  has not been seen failing in the recorded CI history.

### Review checklist

- [ ] A change to the extension list updates the recipe and the `extensions` test together.
- [ ] No extension is removed to make a build pass without the project owner's decision.
- [ ] The CLI's recipe gets the same change.

---

## Exceptions

Only the project owner may drop an extension from the list.

---

## Revision history

- **2026-11-18:** decided, in the PHP foundation design.
- **2026-11-19:** shipped in SpeedySpec/speedyspec-server#4. PHP 8.5.12 built with every extension on macOS arm64 and
  Linux musl arm64; Linux x86_64 was left untested.
- **2026-11-20:** Linux x86_64 fixed in SpeedySpec/speedyspec-server#9.
- **2026-11-21:** recorded as an ADR, and rewritten in this structure.
