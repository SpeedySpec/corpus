# HTTP and static files

`speedyspec-server-presentation`, `speedyspec-server-static`, `speedyspec-server-domain`, `speedyspec-server-app`,
`speedyspec-server`. From the server foundation plan (merged to main before #1).

## The Host header wins over the request's authority, and a second Host is not refused

- **Where:** `speedyspec-server-presentation`, `service.rs` (the request handler) and `forwarded.rs` (`origin`).
- **What happens:** when a request carries both a `Host` header and an authority in its URI, the server routes by
  `Host`; the authority is used only when there is no `Host`. A request with two `Host` headers is served using the
  first one. RFC 9112 §3.2.2 and RFC 9113 §8.3.1 ask for the authority to win and for duplicate `Host` to be refused
  with 400.
- **Suggested fix:** prefer the URI authority when present, and answer 400 to a request with more than one `Host`.

## X-Forwarded-Host and X-Forwarded-Proto take the leftmost value

- **Where:** `speedyspec-server-presentation`, `forwarded.rs`, `apply_x_forwarded`.
- **What happens:** for a trusted proxy, the host and scheme come from the first value in the list. A proxy that
  appends to these headers (instead of replacing them) leaves the client's own value first, so the client chooses
  the host and scheme the server sees. `X-Forwarded-For` is already read from the right.
- **Suggested fix:** read the rightmost value, or the value added by the nearest trusted proxy, as for
  `X-Forwarded-For`.

## Any symlink in a static release fails it

- **Where:** `speedyspec-server-static`, `build.rs`, `collect`.
- **What happens:** the walk skips nothing but directories and regular files; any other entry, including a symlink
  that points inside the release, is an `UnsupportedFile` error, so the release cannot be served. The design only
  forbids symlinks that escape the release (the unpacker already refuses those).
- **Suggested fix:** follow symlinks that resolve inside the release root, and refuse only the rest.

## Mount paths and hosts are not required to be normalized

- **Where:** `speedyspec-server-domain`, `mount.rs`, `MountTable::new`.
- **What happens:** a mount path must start and end with `/`, and a host may not be empty or contain uppercase
  letters, but nothing else is checked. A path like `//admin/` or a host like `example.com:8443` is accepted, but
  requests are normalized before routing, so such a mount never matches anything.
- **Suggested fix:** refuse paths and hosts that are not already in the normalized form (no `//`, `.` or `..`
  segments, no port, no trailing dot).

## A server app with an id other than "server" makes requirements on "server" ignored

- **Where:** `speedyspec-server-app`, `compat.rs`, `check_set`.
- **What happens:** requirements are matched by app id. If the server app is configured as, say, `app.srv`, a release
  that requires `server` names an app that is not in the set, so the requirement is skipped. It is only logged at
  `info` level, and an incompatible set is accepted.
- **Suggested fix:** map a requirement on `server` to whichever app has `kind = "server"`, as `php.runtime` is already
  mapped to the runtime app.

## The 401 for a signed-in mount has no WWW-Authenticate header

- **Where:** `speedyspec-server-presentation`, `service.rs`, the `AccessDecision::Denied` branch.
- **What happens:** a visitor who is not signed in gets `401 sign-in required` without a `WWW-Authenticate` header,
  which HTTP requires on a 401. The design left the challenge to the admin project.
- **Suggested fix:** add the challenge once the admin's sign-in scheme is settled, or answer with a redirect or 403
  instead of 401.

## Only the first Accept-Encoding header line is read

- **Where:** `speedyspec-server-static`, `respond.rs` (`headers.get(header::ACCEPT_ENCODING)`).
- **What happens:** a request that sends `Accept-Encoding` on more than one header line is negotiated from the first
  line only, so a coding listed on a later line (for example `br`) is not used.
- **Suggested fix:** join every `Accept-Encoding` value with commas before choosing.
