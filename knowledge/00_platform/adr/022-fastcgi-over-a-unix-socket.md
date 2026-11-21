# ADR-022: The server talks to PHP workers over FastCGI on a Unix socket

**Status:** Accepted
**Date:** 2026-11-18
**Deciders:** the designer
**Exploration:** [PHP foundation](../speedyspec-server/explorations/2026-11-18-php-foundation.md)
**Related:** [ADR-016](016-rust-runs-as-an-embedded-php-extension.md), [ADR-025](025-supervised-php-worker-pools.md), [ADR-026](026-a-writable-site-directory.md)

---

## Context

The server runs PHP apps in supervised worker processes, the way php-fpm does
([ADR-025](025-supervised-php-worker-pools.md)). The server and its workers are separate processes on the same host,
so they need a protocol between them. The server already had a private socketpair protocol for its keeper process,
which was one option.

### Why this matters

The protocol decides how a worker can be tested and debugged, and what else can talk to it. A private protocol would
need its own tools. Running PHP inside the server would need no protocol at all, but one PHP fatal error or leak
would then take the whole server down.

---

## Decision

The server and its PHP workers speak FastCGI 1.0 over a Unix socket, one socket per worker, with the worker as the
FastCGI responder.

### Rules

1. The only channel between the server and a PHP worker is FastCGI 1.0, in the responder role, over a Unix socket.
2. PHP never runs inside the server process.
3. The worker's socket is private to the server's user (mode `0600`), because FastCGI parameters become `$_SERVER`
   and are trusted.
4. The server's own questions to an app (gate questions, health probes, migrations) are ordinary app endpoints over
   the same FastCGI path. There is no second channel.

### What this looks like

✅ **Correct**

```rust
// crates/speedyspec-php-runtime/src/worker.rs, shortened: the worker is a plain FastCGI responder.
let _ = std::fs::set_permissions(&spec.socket, std::fs::Permissions::from_mode(0o600));
for connection in listener.incoming() {
    let Ok(stream) = connection else { continue; };
    let mut responder = Responder::new(stream);
    while let Ok(Some(head)) = responder.next_request() {
        // run the request's script with head.params as $_SERVER
    }
}
```

❌ **Not permitted**

```rust
// A second, private channel beside FastCGI, or PHP started inside the server process.
let (server_end, worker_end) = UnixStream::pair()?;
send_gate_question(&server_end, &question)?;
```

### Patterns

- **Gateway:** the server's FastCGI client wraps the protocol for the rest of the server.

---

## Consequences

### Positive

1. A worker can be tested and debugged on its own, with any FastCGI tool.
2. A crash, hang or leak in PHP stays in the worker; the server answers 502 or 504 and starts a replacement.
3. When the PHP runtime became its own binary, nothing about the protocol had to change.

### Negative

1. One more hop and a request encoding per request, compared with in-process PHP.
   **Mitigation:** none yet; isolation is the point.
2. The server's own calls share the path with visitors, so revocation is polled rather than pushed.
   **Mitigation:** each pool reserves two workers for the server's own calls; push waits for the live-connections
   design.

### Trade-offs

**We prioritise:**
- Isolation of PHP from the server.
- A standard protocol any tool can speak.

**We accept:**
- A process hop per request.
- Polling for the gate's changes.

---

## Fleet impact

- **speedyspec-server:** sends every PHP request as a FastCGI client. It must not embed PHP.
- **speedyspec-php-runtime:** each worker is a FastCGI responder on its own Unix socket.
- **PHP apps (admin API, front-end API, blog):** receive requests as ordinary FastCGI requests. The admin API must
  answer the gate's questions as normal endpoints. Whether it does yet is unverified.
- **speedyspec-cli:** not affected; it runs PHP in its own process.

---

## Enforcement

### Automated

- `cargo test -p speedyspec-fastcgi --test codec` checks the FastCGI encoding and decoding. It runs in the server's CI
  `test` job. Not seen failing.
- `scripts/check-no-php-in-server.sh` fails with exit 1 if `speedyspec-server` depends on `speedyspec-php`,
  `speedyspec-php-ext`, `ext-php-rs`, `speedyspec-wp-hooks-php` or `speedyspec-container-php`. It runs in CI's `test`
  job. Not seen failing.
- Nothing checks that no second channel is added. That is enforced by review only.

### Review checklist

- [ ] New server-to-app calls are app endpoints over FastCGI, not a new socket or protocol.
- [ ] Worker sockets stay private to the server's user.
- [ ] No PHP is linked into the server binary.

---

## Exceptions

None.

---

## Revision history

- **2026-11-18:** decided, in the PHP foundation design. A private socketpair protocol and PHP inside the server were
  rejected.
- **2026-11-19:** shipped in SpeedySpec/speedyspec-server#4.
- **2026-11-19:** the PHP runtime and admin gate design kept FastCGI as the only channel when the runtime became its
  own binary; the gate's questions and health checks became ordinary app endpoints. Shipped in
  SpeedySpec/speedyspec-server#6.
- **2026-11-21:** recorded as an ADR, and rewritten in this structure.
