# Text formatting

`speedyspec-wp-html-split`, `speedyspec-wp-text*`, `speedyspec-compat` (the `formatting` area). The port of WordPress
7.1.5's `formatting.php` to Rust, kses excepted. It was built from five plans, merged as #33 (the foundation), #38 and
#54 (escaping), #48 (content filters), #59 (sanitizers and URLs) and #68 (the PHP surface).

The first sections cover work that was left out of the port on purpose: functions to port later, the call sites still
to wire up, a speed goal that is not met, and a cheaper check the fast path is waiting on. After those come the Minor
findings from each plan's final whole-branch review, grouped by area. None of them changes the output of a function
on ordinary input: they concern crafted input, the order hooks fire in, test coverage, speed or documentation.

## Functions to port later

- **Where:** `formatting.php`; the PHP stays in use for each of these until it is ported.
- **What happens:** these functions still run WordPress's PHP:
  - `wp_staticize_emoji`, `wp_staticize_emoji_for_email`, `wp_encode_emoji` and `_wp_emoji_list` (a large generated
    table). Only feeds and email use them.
  - `get_url_in_content`, which needs the HTML Tag Processor port (`speedyspec-wp-html`).
  - `wp_sprintf` and `wp_sprintf_l`, a format-string parser over i18n. Only the admin uses them.
- **Suggested fix:** port each one once what it depends on is in Rust. The emoji functions need `_wp_emoji_list`
  generated as a table first.

Three more stay outside this port because they belong to other ports: `esc_sql` goes with the wpdb port,
`wp_pre_kses_block_attributes` goes with the block port (its whole body is `filter_block_content()`), and kses has
its own port.

## Filters are not registered from Rust yet

- **Where:** `register_text_filters` in `speedyspec-wp-text`; WordPress's `default-filters.php`, and the code that
  fires `the_content`.
- **What happens:** the Rust filters exist, and each native is reached through its PHP name. But `default-filters.php`
  still registers them in PHP, and `the_content` is still fired from PHP. So each post's chain goes PHP → Rust →
  PHP for every filter in it.
- **Suggested fix:** when the template port lands, have it call `register_text_filters` and fire `the_content` from
  Rust.

## Speed goals

The numbers come from `bench_php` in `speedyspec-wp-text-php` (`SPEEDYSPEC_TEXT_BENCH=1 cargo test --release -p
speedyspec-wp-text-php --test bench_php`), measured after #68. It runs the natives and WordPress's own
`formatting.php` side by side in the same embedded PHP. The full tables are in the crate's `docs/benchmarks.md`.

| Goal | Target | Now | Met |
|---|---|---|---|
| Every function at least as fast as PHP | 1.0x on every input group | all but three, listed below | no |
| `the_content` chain, whole posts | 5x | 5.1x to 5.2x | yes |
| `the_content` chain, long page | 5x (reported only) | 4.4x to 4.7x | no |

The three functions that fall short:

- **`esc_textarea` on short inputs, about 0.85x.** WordPress's own body calls `get_option( 'blog_charset' )` every
  time, and the native has to make that call through PHP. On a value of a few bytes, that round trip costs more than
  the escaping. On whole posts it is about 3x.
- **`wp_strip_all_tags` on short inputs, 0.68x to 0.76x.** See the next section.
- **`wp_rel_nofollow`, level with PHP (0.94x to 1.08x).** Its time goes on unslashing and reslashing the whole text,
  which PHP also does in C.

`wpautop` is still the chain's largest cost.

## The fast path's rebinding check costs a fifth of a short call

- **Where:** `speedyspec-wp-text-php`, `fast.rs` (`serve_fast`) and `native.rs` (`fast_allowed`).
- **What happens:** four functions have a zero-copy fast path: `convert_chars`, `wp_strip_all_tags`,
  `shortcode_unautop` and `wp_rel_nofollow`. Before taking it, each one checks that its use case still has the text
  API's own handler: `is_alias` and `generation` for its key, so two lookups in the container per call. On a few
  bytes of input those lookups take about a fifth of the call, which puts `wp_strip_all_tags` on short inputs at
  about 0.7x (0.96x to 1.00x without the check). `bench_php` sets the floor for that case at 0.65x.
- **Suggested fix:** add a container-wide generation counter to `speedyspec-container`, bumped on every bind or
  alias. The fast path then compares one number per call. That crate belongs to another port.

## Compat driver and fixtures

From the foundation plan (#33).

### Byte arguments are decoded even when exact bytes are off

- **Where:** `speedyspec-compat`, `driver.php`, the argument decoder.
- **What happens:** a `{"bytes": …}` argument is decoded whether or not the scenario asked for exact bytes. Invalid
  base64 decodes to `false` instead of failing the scenario.
- **Suggested fix:** decode `{"bytes"}` only when `exact_bytes` is on, and fail on invalid base64.

### Malformed encoded values fall back silently

- **Where:** `speedyspec-compat`, the value decoder.
- **What happens:** `{"bytes": <not a string>}` and numeric keys that don't fit an `i64` fall back to something else
  without a word, so a broken fixture can still compare equal.
- **Suggested fix:** reject both with an error that names the scenario.

### Multi-step scenarios are dropped without being counted

- **Where:** `speedyspec-wp-text-corpus`, the case loader and coverage report.
- **What happens:** scenarios with more than one step are left out of the cases and the coverage counts without being
  reported as skipped, so coverage reads higher than it is.
- **Suggested fix:** count them under "skipped" with their own reason.

### A recorded NaN never compares equal

- **Where:** `speedyspec-wp-text-corpus`, value comparison.
- **What happens:** a scenario whose recorded result is NaN can never match, even when the port returns NaN too.
- **Suggested fix:** treat two NaNs as equal when comparing a result with the fixture.

### Diagnostics pile up across steps

- **Where:** `compat/formatting/setup.php`.
- **What happens:** warnings and notices from earlier steps of one scenario are still there when a later step is
  recorded, so a step's diagnostics include its predecessors'.
- **Suggested fix:** clear the diagnostics at the start of each step.

### `ErrorLevel` cannot hold an arbitrary level

- **Where:** `speedyspec-wp-text-contracts`, `ports.rs`, `ErrorLevel`.
- **What happens:** `wp_trigger_error` accepts any integer error level. The enum holds only the named ones, so an
  unusual level can't be carried through.
- **Suggested fix:** add an `Other(i64)` variant.

### The bytes test panics offline

- **Where:** `speedyspec-wp-text-corpus`, `tests/bytes.rs`.
- **What happens:** when WordPress can't be fetched, the test panics instead of being skipped, so the suite fails on
  a machine with no network.
- **Suggested fix:** skip with a message when the release is not available, as the other release tests do.

### The baseline prints MiB/s under an MB/s header

- **Where:** `speedyspec-wp-text-corpus`, `src/bin/php-baseline.rs`.
- **What happens:** the header says MB/s, but the numbers are MiB/s.
- **Suggested fix:** change the header.

### `setup.php` still wraps its body in `if (true) {`

- **Where:** `compat/formatting/setup.php`, line 16.
- **What happens:** a leftover wrapper with no effect.
- **Suggested fix:** remove it.

## Escaping

From the escaping plan (#38, #54), in `speedyspec-wp-text-escape`.

### `EntFlags::from_php` misreads `ENT_HTML5` and drops `ENT_SUBSTITUTE`

- **Where:** `htmlspecialchars.rs`, `EntFlags::from_php`.
- **What happens:** `ENT_HTML5` (48) is read as XML 1, and `ENT_SUBSTITUTE` is ignored, though the doc comment says
  only other bits are ignored. WordPress never passes either, so its own calls are unaffected.
- **Suggested fix:** decode the doctype bits as PHP does, honour `ENT_SUBSTITUTE`, and add corpus cases for both.

### `QuoteStyle` ignores the strings `"double"` and `"single"`

- **Where:** `QuoteStyle`, read from a PHP value.
- **What happens:** WordPress accepts `'double'` and `'single'` as quote styles. The port reads them only as text
  values, not as bytes, so a byte-string argument falls back to the default.
- **Suggested fix:** accept both forms.

### `esc_xml` reads `blog_charset` when nothing needs escaping

- **Where:** `api.rs`, `esc_xml`.
- **What happens:** the port reads `blog_charset` even when the text needs no escaping, and reads it once where
  WordPress reads it once per piece. The output bytes are the same; the number of times the option filter fires is
  not.
- **Suggested fix:** read the option where WordPress does.

### Corpus gaps for `htmlspecialchars` and its decoding

- **Where:** the direct `htmlspecialchars` and `wp_specialchars_decode` corpus.
- **What happens:** no cases for:
  - `&#x0x41;`, numeric overflow, `&#;`, entity names with digits and `&AMP;`;
  - single-byte charsets with `double_encode` false;
  - quote styles given as `true`, `false`, a float, `"2"`, `"0"`, `null` or `[]`.

  The `htmlspecialchars` test also reads only `true` as `double_encode`, not other truthy values.
- **Suggested fix:** add the cases to the generator and re-record.

### No test pins `esc_html`'s filter arguments for invalid UTF-8

- **Where:** `tests/api.rs`.
- **What happens:** for invalid UTF-8, `esc_html` passes the filter an empty string and the original bytes. Nothing
  checks it, so a change could go unnoticed.
- **Suggested fix:** add the test.

### The corpus coverage test's skip check can't fail

- **Where:** `speedyspec-wp-text-corpus`, `tests/coverage.rs`.
- **What happens:** the report skips only functions outside the list, so the assertion that nothing is skipped
  always holds. The real check is the one in `speedyspec-wp-text`'s own coverage test.
- **Suggested fix:** remove the check, or make it check something that can fail.

### `text_error` is defined twice

- **Where:** `speedyspec-wp-text-escape` and `speedyspec-wp-text`.
- **What happens:** both crates define the same conversion from `HookError` to `TextError`.
- **Suggested fix:** one `From<HookError>` impl in `speedyspec-wp-text-contracts`.

### Small drift from the plan

- **Where:** `html401.rs`, `QuoteStyle`, `benches/escape.rs`.
- **What happens:** documentation only:
  - `HTML401_TABLE`'s keys are `&[u8]`, since the characters are bytes.
  - `QuoteStyle::from_php` is private behind `for_specialchars` and `for_decode`.
  - Clippy flags `type_complexity` at `benches/escape.rs:65`; CI doesn't run Clippy on benches.
- **Suggested fix:** note the first two in the crate docs, and give the bench's type an alias.

## Content filters

From the content-filters plan (#48), in `speedyspec-wp-text`.

### Non-UTF-8 shortcode names are mangled on the PCRE path

- **Where:** `texturize/pcre.rs`, `PcreTexturizer`.
- **What happens:** the shortcode pattern is built with `from_utf8_lossy`, so a tag name that is not valid UTF-8 is
  changed before it is matched. Only the PCRE fallback path is affected.
- **Suggested fix:** build the pattern from bytes.

### `wptexturize` returns a number or a boolean unchanged

- **Where:** `TextApi::wptexturize`.
- **What happens:** when nothing changes, an int, float or boolean comes back as it went in. WordPress's `implode`
  always returns a string.
- **Suggested fix:** convert to a string on the way out.

### Hooks fire in a slightly different order

- **Where:** `wptexturize`'s table build, `wptexturize_primes`, `run_wptexturize`, `format_for_editor`.
- **What happens:** the output is the same, but:
  - `wp_spaces_regexp` is read before the translations when the table is built;
  - PHP fires the empty-fragment filter again on every primes call;
  - `run_wptexturize` gets `true` on a reset instead of the previous value;
  - `format_for_editor` reads `blog_charset` for empty text.
- **Suggested fix:** match WordPress's order where a plugin could notice it.

### `wp_trim_words`' charset test misses PCRE's trailing newline

- **Where:** `wp_trim_words`, the charset check.
- **What happens:** PCRE's `$` also matches before a final newline, so WordPress treats `"utf-8\n"` as UTF-8. The
  port does not.
- **Suggested fix:** allow one trailing newline, and add the case.

### The PCRE texturizer's pattern cache is unbounded

- **Where:** `PcreTexturizer`.
- **What happens:** one compiled pattern is kept per set of shortcode tags, with no limit. A site that keeps changing
  its registered shortcodes grows the cache for the life of the worker.
- **Suggested fix:** bound it (a small LRU), or keep only the last set.

### `capital_P_dangit` rebuilds its replacer every call

- **Where:** `capital_p_dangit`.
- **What happens:** a `StrReplace` is built on every call whose text contains "Wordpress". The output is right; it is
  only slower.
- **Suggested fix:** build it once.

### Test gaps in the content filters

- **Where:** `tests/differential.rs`; `FixedTexturizeOverrides` and `FixedShortcodeTags`.
- **What happens:** the differential tests register no `code` shortcode, so the no-texturize-inside-`[code]` path is
  not compared. The two test doubles have no tests of their own.
- **Suggested fix:** register `code` in the differential setup, and add the tests.

## Sanitizers, URLs and links

From the sanitizers-and-URLs plan (#59), in `speedyspec-wp-text`.

### `parse_str` drops the whole variable at the nesting limit

- **Where:** the `parse_str` port.
- **What happens:** at the nesting limit the whole variable is deleted, even when PHP would have stopped first at a
  forbidden index earlier in the key. Only crafted input reaches it.
- **Suggested fix:** check the forbidden index first, as PHP does.

### The Locale port is asked more often than WordPress asks

- **Where:** `sanitize_user`, `remove_accents` (`accents.rs`).
- **What happens:** the port asks for the locale on any non-ASCII input and on every `sanitize_user` call. WordPress
  calls `get_locale()`, and fires its `locale` filter, only for valid non-ASCII UTF-8. The output is the same; the
  number of filter calls is not.
- **Suggested fix:** ask only where WordPress does.

### `sanitize_title` turns an int or null title into a string before its filter

- **Where:** `sanitize_title`, save context.
- **What happens:** PHP's `remove_accents` returns an int or `null` unchanged, so the `sanitize_title` filter gets the
  original type. The port gives the filter a string.
- **Suggested fix:** pass non-strings through `remove_accents` unchanged.

### `links.rs` has its own `is_serialized`

- **Where:** `links.rs`.
- **What happens:** it has a private copy of `is_serialized`, though `speedyspec-php-serialize` exports one.
- **Suggested fix:** use the exported one.

### `smilies_search_pattern` differs for odd smiley keys

- **Where:** `smilies.rs`, `smilies_search_pattern`.
- **What happens:** for an empty smiley key, or numeric-string keys, the published search pattern differs from
  WordPress's `smilies_init`. The engine falls back safely, so conversion is right, but `FixedSmilies` publishes a
  different `$wp_smiliessearch`.
- **Suggested fix:** follow `smilies_init` for those keys.

### `sanitize_user` has a single capture case

- **Where:** `sanitize_user`'s corpus.
- **What happens:** one recorded case. There are none for HTML, entities, `%XX`, runs of whitespace or German
  accents.
- **Suggested fix:** add those literals to the generator.

## The PHP surface

From the PHP-surface plan (#68), in `speedyspec-wp-text-php`.

### `wp_parse_str` with an object gives `['Array' => '']`

- **Where:** `native.rs`, `wp_parse_str`.
- **What happens:** `wp_parse_str( new stdClass )` returns `['Array' => '']`. WordPress throws an `Error`.
- **Suggested fix:** throw when the argument is an object without `__toString`.

### Integer arguments are not read as PHP's `(int)` reads them

- **Where:** `args.rs`, `int_or`.
- **What happens:** `"1e3"`, values that overflow and empty arrays read differently from PHP's `(int)` cast.
- **Suggested fix:** follow PHP's numeric-string rules and its saturation.

### Negative lengths become 0

- **Where:** `utf8_uri_encode`, `_split_str_by_whitespace`.
- **What happens:** a negative length is treated as 0. PHP passes it on, with its own meaning in each function.
- **Suggested fix:** follow WordPress for negative lengths.

### `sanitize_title`'s fallback comes back as a string

- **Where:** `native.rs`, `sanitize_title`.
- **What happens:** an int fallback, such as a post ID, comes back as `"123"`, not `123`.
- **Suggested fix:** return the fallback with its own type.

### The escaping filters receive the converted value

- **Where:** `esc_html`, `esc_attr`, `clean_url`.
- **What happens:** the filter's last argument is the original value after conversion to a string, not the raw value
  the caller passed.
- **Suggested fix:** pass the raw value.

### Slow-path frames show in stack traces

- **Where:** `fast.rs`.
- **What happens:** when a fast function falls back, the stack trace shows a frame named
  `SpeedySpec\WpText\Slow\<name>` under the WordPress name.
- **Suggested fix:** call the native's handler directly instead of through a second registered function.

### String arguments are copied twice

- **Where:** `args.rs`, `native.rs`.
- **What happens:** each string argument is copied once into a `Value` and again when it is read. The output is
  right; it is only slower.
- **Suggested fix:** borrow the bytes from the first copy.
