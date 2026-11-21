# 0024. Static PHP 8.5 with every extension WordPress uses

- **Status:** Accepted, 2026-11-18
- **Decided by:** the project owner ("the embed should include all extensions that WordPress requires"); the list by
  the designer

## Context

The host needs no PHP: PHP is static and embedded (0016). WordPress, its Site Health checks and its plugins expect a
wide set of extensions.

## Decision

Static PHP 8.5, not thread-safe, built with static-php-cli from the CLI's pinned recipe, with the `embed` SAPI.
Builds for macOS arm64 and Linux musl (x86_64 and aarch64). The extensions:

- **Site Health, required:** `hash`, `json`, `pcre`.
- **Site Health, recommended:** `curl`, `dom`, `exif`, `fileinfo`, `imagick`, `mbstring`, `mysqli` with `mysqlnd`,
  `sodium`, `openssl`, `libxml`/`xml`, `xmlreader`, `simplexml`, `zip`, `filter`, `gd` (JPEG, PNG, WebP, AVIF,
  FreeType), `iconv`, `intl`, `zlib`.
- **What WordPress otherwise relies on:** `opcache`, `ctype`, `tokenizer`, `xmlwriter`, `bcmath`, `session`, `pdo` with
  `pdo_mysql`, `pdo_sqlite` and `sqlite3`, and `phar`.
- **Not included:** `mcrypt`, removed in PHP 7.2.

## Consequences

- The list is a hard requirement: an extension that won't build stops the work and goes back to the project owner,
  rather than being dropped quietly.
- CI checks that every listed extension loads, on each target.

## Patterns

None. This is a build decision.
