# PHP runtime and worker pool

`speedyspec-php`, `speedyspec-php-runtime`, `speedyspec-server` (`php/pool.rs`, `php/handler.rs`, `config.rs`), the
`*-php` area crates. From the PHP foundation plan (#4), the PHP runtime and admin gate plan (#6), and one finding
recorded 2026-11-21.

## One panicking area's request_end skips the others

- **Where:** `speedyspec-php`, `area.rs`, `request_shutdown`.
- **What happens:** a single `catch_unwind` wraps the loop over every area. If one area's `request_end` panics, the
  areas after it never run theirs, so their per-request state is carried into the next request.
- **Suggested fix:** catch the panic around each area's `request_end`.

## workers.min = 0 retires and respawns the last worker

- **Where:** `speedyspec-server`, `php/pool.rs`, `reap`.
- **What happens:** with `min = 0`, the reaper retires the last idle worker after `idle_timeout`, then sees fewer
  than one worker (`min.max(1)`) and starts a new one. The pool restarts a worker every `idle_timeout` while idle.
- **Suggested fix:** retire only above `min.max(1)`, or let `min = 0` really mean no idle workers.

## A worker that died on its own is logged as "the release is being retired"

- **Where:** `speedyspec-server`, `php/pool.rs`, `give_back`.
- **What happens:** a worker that answered its request but then exited is not crashed, timed out or due for
  recycling, so it falls into the last branch, which logs `php_worker_recycled` with the reason "the release is being
  retired". The log misleads whoever is looking for why the worker went away.
- **Suggested fix:** give that case its own reason, such as "exited after answering".

## give_back waits for a killed worker while holding the pool lock

- **Where:** `speedyspec-server`, `php/pool.rs`, `give_back` (the `Crashed` and `TimedOut` branches).
- **What happens:** the worker is killed and then waited for (`child.wait()`) with the pool's state mutex held. Every
  other request that needs a worker waits too, for as long as the process takes to exit.
- **Suggested fix:** take the worker out under the lock, then kill and wait for it after releasing the lock.

## Non-ASCII response header values are dropped

- **Where:** `speedyspec-server`, `php/handler.rs`, `parse_head`.
- **What happens:** header values from PHP are built with `HeaderValue::from_str`, which refuses bytes outside
  visible ASCII. A header with a UTF-8 value (for example a `Content-Disposition` file name) is silently left out of
  the response.
- **Suggested fix:** use `HeaderValue::from_bytes`, which accepts obs-text.

## Static files under a PHP app lack Range, ETag and Last-Modified

- **Where:** `speedyspec-server`, `php/handler.rs`, `serve_static` and `content_type`.
- **What happens:** files served from a PHP app's directory are streamed whole, with no range support and no
  validators, so clients cannot resume or revalidate them. The MIME list has no `mp4`, `webm`, `mp3` or `wasm`; those
  are sent as `application/octet-stream`, which breaks `WebAssembly.instantiateStreaming`.
- **Suggested fix:** reuse the static crate's range and conditional handling, and add the missing types.

## HTTP_ header names can collide, and Proxy is passed through

- **Where:** `speedyspec-server`, `php/handler.rs`, where request headers become CGI parameters.
- **What happens:** `X_Foo` and `X-Foo` both become `HTTP_X_FOO` and are joined into one value. A request's `Proxy`
  header becomes `HTTP_PROXY`, which some PHP HTTP clients read as their proxy setting (httpoxy).
- **Suggested fix:** drop header names that contain `_`, as nginx does by default, and never pass `Proxy`.

## The web request pointer is not cleared if something panics

- **Where:** `speedyspec-php`, `web.rs`, `begin` and `end`.
- **What happens:** `begin` stores a raw pointer to the request in a thread-local, and `end` clears it. A panic
  between the two leaves the pointer set to a request that no longer exists.
- **Suggested fix:** clear it in a guard's `Drop`, so it is cleared on every path.

## Env file values keep their quotes

- **Where:** `speedyspec-server`, `config.rs`, `env_lines` (used by `worker_env` for `.env` and `env_file`).
- **What happens:** a line `KEY="value"` gives the worker `"value"` with the quotes, unlike the dotenv format most
  people expect.
- **Suggested fix:** strip one pair of matching quotes, or document that values are taken as written.

## The facades are compiled with eval on every request

- **Where:** `speedyspec-php`, `runtime.rs` (the preload, run again at the start of each request); the `*-php`
  crates' facade sources (for example `speedyspec-container-php`).
- **What happens:** the PHP facades (about 850 lines) are evaluated with `eval` at the start of every request. This
  costs compile time per request. It should be measured before WordPress traffic runs on it.
- **Suggested fix:** measure first; if it matters, cache the compiled classes (opcache or an immutable class table).

## The PHPUnit phar is pinned by trust on first use

- **Where:** `tests/php/phpunit.json` and the tests that fetch PHPUnit (for example
  `crates/speedyspec-wp-db-php/tests/wordpress.rs`).
- **What happens:** the phar is checked against a SHA-256 recorded when it was first fetched. Its PGP signature is
  never checked, so the pin is only as good as that first download.
- **Suggested fix:** verify the PGP signature once when updating the pin.

## The worker parses the whole spec before checking the contract major

- **Where:** `speedyspec-php-runtime`, `main.rs` (`serde_json::from_str::<WorkerSpec>`) and `worker.rs`
  (`spec.check()`).
- **What happens:** a server with a newer contract that changed the spec's shape makes the worker fail with a JSON
  parse error, instead of the clear contract-major refusal that names both versions.
- **Suggested fix:** read the contract version first, refuse a different major, then parse the rest.

## The spec says the server's suite passes with no static PHP

- **Where:** `docs/superpowers/specs/2026-11-19-speedyspec-php-runtime-and-admin-gate-design.md` ("No PHP in the
  server"); `.github/workflows/ci.yml`.
- **What happens:** the spec says the server builds and passes its suite with no static PHP present. The build does,
  but the server's own tests start the PHP runtime's workers, so CI runs them only in the `php` job, where PHP is
  built.
- **Suggested fix:** record that in the spec.

## speedyspec-php cannot be published to crates.io as it is

- **Where:** `crates/speedyspec-php/build.rs`.
- **What happens:** the build script reads `<workspace>/php-version` and, unless `SPEEDYSPEC_PHP_PREFIX` is set,
  links the static PHP from `<workspace>/.php-build/buildroot`. Both are outside the crate, so a published copy of the
  crate would not build. Publishing is planned.
- **Suggested fix:** ship the pinned version inside the crate, and require the PHP prefix from an environment
  variable (or a documented default) when built outside the workspace.
