# KSES

`speedyspec-wp-kses*`, `speedyspec-wp-html-syntax`. The port of WordPress 7.1.5's kses to Rust, merged as #50, with
the CI fix in #52, the token-map switch in #53 and the speed-up in #58.

The first six findings are the Minor ones from the final whole-branch review. None lets unsafe HTML through: the first
three are stricter than PHP, and the rest are speed, documentation or a missing bound. The last section records the
speed goals that are still not met.

## A null rule counts as present

- **Where:** `speedyspec-wp-kses`, the directive check on an allowed attribute's rule.
- **What happens:** a rule whose value is `null` is treated as present. PHP's `isset()` treats it as absent, so the
  port can reject an attribute that WordPress keeps. Stricter than PHP, never looser.
- **Suggested fix:** treat a `null` rule as absent, as `isset()` does, and add the case to the differential corpus.

## `wp_kses( $x, true )` is not the `post` context

- **Where:** `speedyspec-wp-kses`, the context lookup.
- **What happens:** the context becomes the string `"1"`, which matches no allowlist, so everything is stripped.
  WordPress's loose `switch` matches `true` against `'post'` and uses the post allowlist. Stricter than PHP.
- **Suggested fix:** map a boolean `true` context to `post`, as PHP's loose comparison does.

## An empty scalar from `safe_style_css` strips all CSS

- **Where:** `speedyspec-wp-kses`'s `safecss`, reading the `safe_style_css` filter's result.
- **What happens:** a filter that returns an empty scalar, such as `''` or `false`, leaves no allowed properties, so
  every declaration is removed. WordPress returns the CSS unchanged in that case. Stricter than PHP.
- **Suggested fix:** follow PHP's handling of a non-array result, and add a compat scenario that records it.

## An unset or non-array watched global defeats the allowlist cache

- **Where:** `speedyspec-wp-kses`'s `globals.rs`, the `PhpAllowedGlobals` watcher.
- **What happens:** when `$allowedposttags` or another watched global is unset or not an array, the cache can't tell
  whether it changed, so the allowlist is rebuilt on every call. The output is right; only the speed suffers.
- **Suggested fix:** remember the "unset or not an array" state as its own cache key.

## `AttributeEscaper`'s duty is not documented

- **Where:** the `AttributeEscaper` port in `speedyspec-wp-kses-contracts`.
- **What happens:** the safety of kept attribute values depends on the escaper encoding `"`, `<`, `>`, `'` and `&`.
  The trait doesn't say so, so a new adapter could leave one out without any test noticing.
- **Suggested fix:** state the duty in the trait's doc comment, and add a contract test that every adapter must pass.

## Deeply nested filter arrays are read without a depth bound

- **Where:** `speedyspec-wp-kses`'s `convert.rs`, `rule_value` and `value_of`.
- **What happens:** both recurse into the arrays a filter returns, with no depth limit and no stack growth (such as
  `stacker`). A filter that returns an array nested tens of thousands of levels deep can overflow the stack and
  crash the worker. Everywhere else, kses bounds its own work.
- **Suggested fix:** cap the depth, past which the rule fails closed, or grow the stack with `stacker`. Add a test
  that nests deeply.

## Speed goals

The design set three goals. Numbers come from `cargo bench -p speedyspec-wp-kses`, measuring thread CPU time (best of
5 rounds on a quiet machine), after #58.

| Goal | Target | Now | Met |
|---|---|---|---|
| `wp_kses_post` on snippets | 250 MB/s | 341–365 MB/s | yes |
| `wp_kses_post` on a full page | 500 MB/s | 363–389 MB/s | no, about 1.3× short |
| The PHP twin, `speedyspec-wp-kses-php`, against legacy PHP kses | 5× faster | 5.2–5.3× | yes, barely |

Before #58, both Rust paths measured about 190–210 MB/s.

### Full page: what's left

- **Where:** mostly `speedyspec-wp-hooks`; the rest is spread thin across `speedyspec-wp-kses`.
- **What happens:** the largest single cost, about 9–12%, is `HookApi::skip_if_empty`. Each time kses counts a filter
  run that has no listeners, it does three map lookups. kses must make those calls to keep the hook run counts the
  same as PHP's.
- **Suggested fix:** make the count-only path in the hooks crate a single lookup. The crate belongs to the Hook API
  port. Then profile again: no other single piece stands out.

### The PHP twin: what's left

- **Where:** `speedyspec-wp-kses-php`'s `PhpEscaper`.
- **What happens:** it calls PHP's `esc_attr` once per kept attribute, which costs most of the twin's remaining time.
- **Suggested fix:** use the native `speedyspec_wp_text_escape::EscapeApi::esc_attr` once text-formatting's plan 5
  connects it to the PHP crates.
