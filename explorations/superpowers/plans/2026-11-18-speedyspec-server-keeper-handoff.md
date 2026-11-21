# SpeedySpec Server Keeper and Socket Handoff Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or
> superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** `speedyspec-server run` becomes a small, long-lived **keeper** that owns the sockets and runs the server in
a child **generation**. In managed mode the server can then upgrade and roll back its own binary by socket handoff:
- no dropped HTTP requests;
- a trial period, with automatic fall-back;
- group switches that include the server;
- `reload`;
- certificate reload on `SIGHUP`;
- the keeper re-executing itself.

In both modes the keeper restarts a crashed generation.

**Architecture:**
- **`speedyspec-server-handoff`** (the crate the spec names): the versioned keeper–generation protocol, as JSON
  lines over a Unix socketpair, with listening sockets passed alongside as `SCM_RIGHTS` (via `rustix`).
- **The binary runs in two roles.** `run` is the keeper. `serve` (hidden) is a generation, started only by a keeper.
- **The keeper:**
  - binds every listener and holds `server.lock`;
  - serves `keeper.sock` and forwards each request to the active generation;
  - owns the trial timer, and writes the server's trial fields and the pending group in `state.json`.
- **Generations:**
  - run everything plan 2 built;
  - expose a `Serving` controller that can stop accepting, hand back UDP and resume;
  - ask the keeper for an upgrade when a server release is switched to.
- **Server binaries** are releases like any other. They live in `releases/server/<version>/speedyspec-server`, and a
  binary's manifest is read by running it with `manifest`.

**Tech Stack:** Plan 2's stack, plus `rustix` (socketpair, `sendmsg`/`recvmsg` with `SCM_RIGHTS`, clearing
close-on-exec, `execve`) and `libc` only where `rustix` lacks a call.

**Spec:** `docs/superpowers/specs/2026-11-17-speedyspec-server-core-design.md`, especially "Deployment modes",
"Socket handoff", "Who runs release operations", "A group switch that includes the server", "Control" and "Testing".

**This is plan 3 of 4.**
- Plan 2 is [SpeedySpec/speedyspec-server#1](https://github.com/SpeedySpec/speedyspec-server/pull/1). Branch from
  `main` once it is merged; until then, stack on `feat/managed-releases` and say so in the PR.
- Plan 4 (cluster: leadership, member files, two-step switch, pod-by-pod rollout) builds on this plan's trial and
  pending-group state.

## Global Constraints

- Work in `packages/speedyspec-server/`. Do not push or open a PR without Jacob's go-ahead.
- Rust 1.99, edition 2024, default rustfmt. `cargo fmt --all` before every commit. Conventional Commits, each ending
  with `Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>`.
- Salvo appears only in `-presentation` and the binary. Contracts live in `-domain`. `-handoff` holds the wire
  protocol and its framing, and has no Salvo dependency.
- TLS stays on `ring`: `cargo tree -i aws-lc-sys` prints nothing.
- Unix only, and every test passes on macOS arm64 and on Linux musl (the Docker command from plan 2).
- The keeper serves no traffic and never exits during a handoff.
- Immutable mode never writes to disk. In immutable mode the keeper only restarts crashed generations: no handoff,
  and `reload` is refused.
- Every refusal names what was refused and why. Logs are JSON lines on stderr, and each handoff step is its own event.
- Test hooks are compiled only with the `test-hooks` Cargo feature (off by default). Tests run with
  `cargo test --workspace --features speedyspec-server/test-hooks`, and CI is updated to match.

Spec values used across tasks (the config keys are my calls, under `[handoff]`):

| Value | Default |
|---|---|
| `handoff.ready_timeout` | 30 s |
| `handoff.drain_timeout` | 5 minutes |
| `handoff.trial_period` | 2 minutes |
| Crash restart back-off | 1 s, doubling to 1 minute |
| Under systemd | `Type=notify`; `READY=1` once the first generation is ready |

## Review Focus

Each line has a test in the task named in brackets.

1. **The keeper is killed with SIGKILL.** Its generation notices that the channel has closed and exits within 2 s, so
   systemd or the container runtime restarts cleanly and nothing keeps serving unsupervised. [Task 6]
2. **A new binary that exits right after starting, or never becomes ready.** The old generation keeps serving without
   interruption, the version is recorded as failed, and the keeper does not start it again. [Task 7]
3. **An upgrade asked for while another upgrade is on trial**, or while a generation is draining. It is refused with
   a message naming the version on trial. [Task 7]
4. **SIGTERM in the middle of an upgrade.** Both generations drain and exit, the keeper exits 0, and no process is left
   running. [Task 7]
5. **A leftover `keeper.sock` (or plan 2's `control.sock`) from a crashed run.** A new keeper replaces a dead socket
   and refuses a live one. Generations bind no socket files: they answer control requests over their channel. [Task 6]

## File Structure

```
crates/speedyspec-server-domain/src/state.rs    + ServerState, PendingGroup (serde defaults)
crates/speedyspec-server-domain/src/contracts.rs + BinaryInspector
crates/speedyspec-server-handoff/               src/lib.rs protocol.rs channel.rs   tests/channel.rs
crates/speedyspec-server-app/src/releases.rs    + server releases, pending groups, trial outcome helpers
crates/speedyspec-server/build.rs               exports SPEEDYSPEC_TARGET (the Rust target triple)
crates/speedyspec-server/src/serving.rs         Serving controller (replaces listeners::serve's body)
crates/speedyspec-server/src/generation.rs      the `serve` role
crates/speedyspec-server/src/keeper.rs          the `run` role: spawn, supervise, forward, upgrade, trial, re-exec
crates/speedyspec-server/src/binary.rs          BinaryInspector impl, first-start registration, test hooks
crates/speedyspec-server/src/notify.rs          sd_notify
crates/speedyspec-server/tests/                 keeper.rs, upgrade.rs, handoff_under_load.rs, reload.rs
docs/handoff.md                                 what Task 1 proved, and the message flow
```

---

### Task 1: Prove what the handoff needs from Salvo, quinn and the kernel

This is a proof, like plan 1's Task 1. The handoff depends on four things none of the code does yet. This task
decides how each works, or stops.

**Files:**
- Create: `docs/handoff.md`
- Create: `crates/speedyspec-server/tests/handoff_proof.rs`, with its helpers in `tests/support/`

**Interfaces:**
- Produces findings only. The production code comes in Tasks 4–5, and it must follow what this task records.

- [ ] **Step 1: Write four proof tests** in `tests/handoff_proof.rs`, each using plan 1's
  `listeners`/`support::{certs,h3}`:
  - `tcp_stops_accepting_then_resumes`:
    - Serve on a pre-bound `TcpListener`, then stop accepting while one slow request (1 s) is in flight; it completes.
    - New connections queue in the backlog, and are served once serving resumes on a `try_clone` of the same
      listener.
    - Assert zero failed requests.
  - `two_processes_share_a_listener`:
    - Pass a `TcpListener` and a `UdpSocket` to a child process (the test binary re-run with an env var) over a
      `rustix` socketpair with `SCM_RIGHTS`.
    - The child serves HTTP/1.1 on the TCP socket while the parent still holds it. After the parent stops accepting,
      every new request reaches the child.
  - `quic_endpoint_hands_over`:
    - The parent serves HTTP/3, then closes its quinn endpoint with application code `0x5353` (upgrading) and drops
      it, without closing the socket's other copies.
    - The child builds a new endpoint over its copy of the same `UdpSocket` and answers an HTTP/3 request.
    - An HTTP/3 connection open on the parent receives a connection close carrying code `0x5353`.
  - `quic_endpoint_resumes`: after the parent's endpoint is closed, a new endpoint in the parent over a `try_clone` of
    the socket answers HTTP/3 again. This is the resume path of a failed trial.
- [ ] **Step 2: Run** `cargo test -p speedyspec-server --test handoff_proof` → FAIL.
- [ ] **Step 3: Make them pass** with the smallest code that does it, inside the test file and its support module.
  - If Salvo's `stop_graceful` cannot be paired with a fresh `Server` on a cloned listener, record what works instead
    (for example, an acceptor wrapper that can be paused).
  - If a QUIC endpoint cannot be rebuilt on a socket that another endpoint has used, stop and report to Jacob. The
    resume path depends on it.
- [ ] **Step 4: Run** on macOS and on Linux musl → PASS on both.
- [ ] **Step 5: Write `docs/handoff.md`:**
  - how each of the four works;
  - the `rustix` calls used;
  - any Salvo or quinn behaviour later tasks must respect, such as whether `stop_graceful` sends HTTP/2 and HTTP/3
    `GOAWAY`;
  - the message sequence of an upgrade, a resume and a crash restart, as in the spec's "Socket handoff" steps 1–6.
- [ ] **Step 6: Commit.** `test: prove pause/resume, cross-process sockets and QUIC endpoint handover`

---

### Task 2: Domain: server trial state, pending groups, and inspecting binaries

**Files:**
- Modify: `crates/speedyspec-server-domain/src/state.rs`, `src/contracts.rs`, `src/lib.rs`
- Test: unit tests in `state.rs`

**Interfaces:**
- Produces, `state.rs`:
  - `ReleaseState.server: ServerState`.
  - `pub struct ServerState { pub confirmed: Option<Version>, pub trial: Option<Version>, pub failed: BTreeMap<Version, String> }`.
  - `ReleaseState.pending_group: Option<PendingGroup>`.
  - `pub struct PendingGroup { pub server: (Version, Version), pub apps: Vec<(AppId, Option<Version>, Version)> }`.
    The tuples are (previous, target) for the server, and (app, previous, target) for each app.
  - Every field defaults, so plan 2 state files still load.
- Produces, `manifest.rs`: `ReleaseManifest.protocol: Option<u32>`, the keeper protocol a server binary speaks. It is
  allowed only for kind `server` and serialized only when set, because the manifest rejects unknown fields. Task 9
  reads it.
- Produces, `contracts.rs`:
  - `pub trait BinaryInspector: Send + Sync { fn manifest(&self, binary: &Path) -> Result<ReleaseManifest, String>; }`
- `ServerState` lives apart from `apps[server]`. `apps[server].kept` keeps listing the kept server binaries, so plan 2's
  prune and `kept` logic apply to the server unchanged. `active` for the server app is the running generation's
  version.

- [ ] **Step 1: Write the failing tests:**
  - `plan2_state_file_still_loads`: a plan 2 `state.json` literal → `server` is the default, `pending_group` is `None`.
  - `server_state_round_trips`.
  - `protocol_only_on_server_manifests` (a `static` manifest with `protocol` is refused).
  - `pending_group_round_trips`.
- [ ] **Step 2: Run** `cargo test -p speedyspec-server-domain` → FAIL.
- [ ] **Step 3: Implement.**
- [ ] **Step 4: Run** → PASS.
- [ ] **Step 5: Commit.** `feat(domain): server trial state, pending groups and the binary inspector contract`

---

### Task 3: Server releases in the release manager

**Files:**
- Modify: `crates/speedyspec-server-app/src/releases.rs`, `src/lib.rs`
- Modify: `crates/speedyspec-server-infra-fs/src/store.rs` (add `commit_binary`)
- Modify: `crates/speedyspec-server-infra-memory` (add `FakeInspector`)
- Test: `crates/speedyspec-server-app/tests/server_releases.rs`, `crates/speedyspec-server-infra-fs/tests/store.rs`

**Interfaces:**
- Consumes: Task 2; plan 2's `ReleaseManager`, `fetch`, `unpack_and_keep`, `switch_locked` and `commit_state`.
- Produces:
  - `ReleaseManager::new(…)` takes one more argument, `inspector: Arc<dyn BinaryInspector>`, and the target triple as
    `target: String`.
  - `ReleaseStore::commit_binary(&self, app: &AppId, version: &Version, artifact: &Path) -> Result<PathBuf, StoreError>`.
    It moves the verified artifact to `releases/<app>/<version>/speedyspec-server` with mode 0755, and returns that
    path.
  - Installing a server release:
    1. Take the index artifact for `target`, not `any`.
    2. Download, checksum and verify as for any release.
    3. `commit_binary`.
    4. `inspector.manifest(path)`. It must be `kind = server` with this version, and the release is refused and
       recorded as failed otherwise.
    5. Push the version onto `apps[server].kept`.
    - `ServerNeedsKeeper` is gone from `install`.
  - `ReleaseManager::begin_server_switch(&self, targets: &[(AppId, Version)]) -> Result<ServerUpgrade, ReleaseError>`,
    used when the targets include the server:
    - Check compatibility over the resulting set.
    - Refuse while `server.trial` or `pending_group` is set: `UpgradeInProgress(Version)`.
    - Write `pending_group` (previous and target for every member) and `server.trial = target`, then save.
    - Return `pub struct ServerUpgrade { pub version: Version, pub binary: PathBuf }`.
    - Nothing activates yet.
  - `switch` with a server target returns `ReleaseError::ServerSwitchViaKeeper`. Only the control backend calls
    `begin_server_switch`, and it then asks the keeper (Task 8).
  - `ReleaseManager::apply_pending_group(&self, own_version: &Version) -> Result<(), ReleaseError>`:
    - Called in a new generation's start-up, before it reports ready.
    - When `pending_group.server.1 == own_version`, it switches the group's apps to their targets (prepare and
      activate). The server's own `active` is set to `own_version`.
  - `ReleaseManager::revert_pending_group(&self, failed: &Version, reason: &str) -> Result<(), ReleaseError>`:
    - Puts each app's `active` back to its previous version and reactivates those releases.
    - Clears `pending_group` and `server.trial`, and records `server.failed[failed] = reason`.
    - It is called by a resumed generation, or by a fresh fall-back generation in its `boot`.
  - `ReleaseManager::boot` (plan 2) gains a first step: when `pending_group` is set and its server target is not this
    binary's version, it runs `revert_pending_group(target, "trial failed")` before anything else.
  - `pub fn confirm_server(state: &dyn StateStore, version: &Version) -> Result<(), StateError>` (a free function). It
    sets `server.confirmed`, clears `server.trial` and `pending_group`, and saves. The keeper calls it at the end of a
    trial, because the keeper has no `ReleaseManager`.
  - `pub fn register_running_binary(store: &dyn ReleaseStore, state: &dyn StateStore, app: &AppId, version: &Version, exe: &Path) -> Result<PathBuf, ReleaseError>`:
    - First start in managed mode.
    - If no server release is kept, it copies `exe` in as `version`, marks it kept and confirmed, and saves.
    - It returns the confirmed binary's path.
  - New error variants: `UpgradeInProgress(Version)`, `ServerSwitchViaKeeper`, `Inspect(String)`, and
    `NoArtifactForTarget(AppId, Version, String)`.

- [ ] **Step 1: Write the failing tests:**
  - `installs_server_binary_for_target`: the index lists `x86_64-unknown-linux-musl` and `aarch64-apple-darwin`
    artifacts, the manager's target picks one, and the kept file is executable.
  - `server_binary_with_wrong_manifest_refused_and_failed` (`FakeInspector` returns kind `static`).
  - `missing_target_artifact_refused`.
  - `begin_server_switch_records_pending_group_and_trial`.
  - `begin_server_switch_refused_during_trial` (Review Focus 3).
  - `begin_server_switch_refuses_incompatible_set`.
  - `apply_pending_group_switches_apps_for_own_version`.
  - `apply_pending_group_ignores_other_version`.
  - `boot_reverts_group_of_a_failed_trial`: the state has a pending group for server 2.0.0, and boot runs as 1.0.0 →
    the apps are back on their previous versions, `server.failed[2.0.0]` is set, and the group is cleared.
  - `confirm_server_clears_trial`.
  - `register_running_binary_is_idempotent`.
  - infra-fs: `commit_binary_sets_executable`.
- [ ] **Step 2: Run** `cargo test -p speedyspec-server-app -p speedyspec-server-infra-fs` → FAIL.
- [ ] **Step 3: Implement.**
- [ ] **Step 4: Run** `cargo test --workspace` → PASS (plan 2's tests are updated for the new `new` arguments).
- [ ] **Step 5: Commit.** `feat(app): server releases, pending groups and trial bookkeeping`

---

### Task 4: `speedyspec-server-handoff`: the keeper–generation protocol

**Files:**
- Create: `crates/speedyspec-server-handoff/` (deps: serde, serde_json, rustix with the `net` and `fs` features,
  thiserror), `src/lib.rs`, `src/protocol.rs`, `src/channel.rs`
- Test: `tests/channel.rs`

**Interfaces:**
- Produces, `protocol.rs`:
  - `pub const PROTOCOL: u32 = 1;`
  - `pub enum ToGeneration`:
    - `Hello { protocol: u32, config: PathBuf, sockets: Vec<SocketInfo>, role: GenerationRole }`, with one fd per
      `SocketInfo`, in order.
    - `StartUdp`
    - `Drain`
    - `Resume { failed: Option<Version> }`
    - `ReloadCerts`
    - `Shutdown`
    - `Control(String)`: one control request line, forwarded from `keeper.sock`.
  - `pub enum ToKeeper`:
    - `Ready { version: Version, protocol: u32 }`
    - `UdpReleased`
    - `Upgrade { version: Version, binary: PathBuf }`
    - `Reload`
    - `ControlReply(String)`
    - `Exiting`
  - `pub struct SocketInfo { pub kind: SocketKind /* Tcp | Udp */, pub address: SocketAddr, pub tls: Option<(PathBuf, PathBuf)>, pub http3: bool }`
  - `pub enum GenerationRole { Managed, Immutable }`
  - Every message is serde, externally tagged, with `#[serde(other)]`-style tolerance of unknown variants in receivers.
    Unknown messages are logged and ignored, so a newer keeper and an older generation, or the reverse, keep working.
    The protocol changes only by adding messages; the spec says every keeper speaks every version that has shipped.
- Produces, `channel.rs`:
  - `pub struct Channel` over one end of a `UnixStream` socketpair.
  - `Channel::pair() -> io::Result<(Channel, OwnedFd)>`: the second value is the child's end, to be placed on fd 3.
  - `Channel::from_fd(fd: OwnedFd) -> io::Result<Channel>`
  - `send(&self, message: &impl Serialize, fds: &[BorrowedFd]) -> io::Result<()>`
  - `recv<T: DeserializeOwned>(&self) -> io::Result<Option<(T, Vec<OwnedFd>)>>`: `None` on EOF.
  - Framing: one JSON line per message. Its fds ride on the `sendmsg` that carries the line's first byte.
  - Async: `Channel::into_async(self) -> AsyncChannel`, over tokio's `UnixStream`, with the same `send` and `recv` as
    `async fn`.

- [ ] **Step 1: Write the failing tests:**
  - `round_trips_every_message`.
  - `passes_listening_sockets`: send a bound `TcpListener` and `UdpSocket`; the receiving end accepts a connection on
    the received fd, and receives a datagram on the other.
  - `unknown_message_is_skipped` (a raw `{"Future":{}}` line, then `Shutdown` → `recv` yields `Shutdown`).
  - `eof_is_none`.
  - `fds_are_close_on_exec_by_default`.
- [ ] **Step 2: Run** `cargo test -p speedyspec-server-handoff` → FAIL.
- [ ] **Step 3: Implement** with `rustix::net::{sendmsg, recvmsg, SendAncillaryBuffer, RecvAncillaryBuffer, SendAncillaryMessage::ScmRights}`.
- [ ] **Step 4: Run** → PASS on macOS and Linux.
- [ ] **Step 5: Commit.** `feat(handoff): versioned keeper–generation protocol with fd passing`

---

### Task 5: The `Serving` controller

**Files:**
- Create: `crates/speedyspec-server/src/serving.rs`
- Modify: `src/listeners.rs` (`serve` becomes a thin wrapper over `Serving`), `src/lib.rs`
- Test: `crates/speedyspec-server/tests/serving.rs`

**Interfaces:**
- Consumes: Task 1's findings (`docs/handoff.md`), plus plan 1's `BoundTcp` / `BoundUdp` / `TlsFiles`.
- Produces:
  - `pub struct Serving`
  - `Serving::start(tcp: Vec<BoundTcp>, udp: Vec<BoundUdp>, router: Arc<Router>, read_udp: bool) -> anyhow::Result<Serving>`
    - It keeps a `try_clone` of every socket, so it can resume.
  - `Serving::start_udp(&self)`: build the QUIC endpoints. Idempotent.
  - `Serving::stop_accepting(&self)`:
    - Stop accepting TCP, and send HTTP/2 and HTTP/3 `GOAWAY`.
    - Close every QUIC endpoint with application code `0x5353` and the reason `server upgrading, reconnect`, then drop
      it.
    - It returns once UDP is released.
  - `Serving::resume(&self)`: accept TCP again, and rebuild the QUIC endpoints on the cloned sockets.
  - `Serving::reload_certs(&self) -> anyhow::Result<()>`:
    - Re-read every listener's PEM files, and feed them into the TLS listeners' config streams and into
      `Endpoint::set_server_config`.
    - A bad file is logged, and the old certificate is kept.
  - `Serving::wait_idle(&self, timeout: Duration)`: returns when no connection is open, or when `timeout` passes, and
    then force-closes what remains.
  - `pub const UPGRADE_CLOSE_CODE: u32 = 0x5353;`
- `Alt-Svc` (left over from plan 1): every response on a TLS TCP listener whose `http3 = true` carries
  `Alt-Svc: h3=":<port>"; ma=86400`. It is added by a Salvo hoop in the router `Serving` builds, which compares the
  request's local port.

- [ ] **Step 1: Write the failing tests:**
  - `stop_then_resume_serves_again` (h1, h2c, HTTPS and h3; zero failures across the pause, with requests sent during
    the pause served after resume).
  - `inflight_request_finishes_after_stop`.
  - `open_h3_connection_gets_upgrade_close_code` (a `support::h3` helper variant that keeps the connection open and
    reports the close code).
  - `wait_idle_forces_close_after_timeout`.
  - `reload_certs_serves_new_certificate` (regenerate the leaf with a different serial, then `reload_certs`; a new TLS
    connection sees the new serial).
  - `bad_cert_keeps_old`.
  - `alt_svc_on_tls_with_http3_only` (present on HTTPS with h3; absent on cleartext, and on TLS without h3).
  - Plan 1's `listeners_proof` still passes through the wrapper.
- [ ] **Step 2: Run** `cargo test -p speedyspec-server --test serving` → FAIL.
- [ ] **Step 3: Implement**, following `docs/handoff.md`.
- [ ] **Step 4: Run** `cargo test --workspace --features speedyspec-server/test-hooks` → PASS.
- [ ] **Step 5: Commit.** `feat: serving controller that pauses, hands back UDP, resumes and reloads certificates`

---

### Task 6: The keeper and generations, without upgrades

**Files:**
- Create: `crates/speedyspec-server/src/keeper.rs`, `src/generation.rs`, `src/notify.rs`, `src/binary.rs`
- Modify: `src/main.rs` (`run` → keeper; hidden `serve`), `src/managed.rs` (the server lock moves to the keeper),
  `src/control.rs` (the generation socket path, and the keeper's status merge), `src/config.rs` (`[handoff]` keys)
- Create: `crates/speedyspec-server/build.rs` (exports `SPEEDYSPEC_TARGET` from cargo's `TARGET`)
- Test: `crates/speedyspec-server/tests/keeper.rs`; plan 1 and 2 integration tests keep passing unchanged

**Interfaces:**
- Consumes: Tasks 3–5. `managed::start` now passes `ProcessInspector` and `env!("SPEEDYSPEC_TARGET")` to
  `ReleaseManager::new`.
- Produces:
  - `keeper::run(config_path: &Path) -> Result<(), String>`. In order:
    1. Load the config.
    2. Take `<run_dir>/server.lock` (moved here from `managed::start`; a generation never takes it).
    3. Bind every listener.
    4. Serve `<run_dir>/keeper.sock`. A live socket is refused, a dead one replaced.
    5. In managed mode, `register_running_binary` the current executable and use the confirmed binary; in immutable
       mode use the current executable.
    6. Start generation 1 and wait for `Ready` (`ready_timeout`).
    7. Send `StartUdp`.
    8. `sd_notify("READY=1")`.
    9. Supervise.
  - Spawning a generation:
    - `Command::new(binary).arg("serve")`, with the child's channel end placed on fd 3 through `pre_exec` + `dup2`, and
      `SPEEDYSPEC_CHANNEL_FD=3`.
    - Stdout and stderr are inherited.
    - Then `Hello` carries the sockets.
  - Supervision:
    - **A generation exits outside an upgrade** → log `generation_exited` with its status. Start a new one from the
      confirmed binary, with back-off of 1 s doubling to 60 s, reset after 60 s of uptime.
    - **`SIGTERM`/`SIGINT`** → `Drain` every generation, wait for each `Exiting` (or `drain_timeout`), then exit 0.
    - **`SIGHUP`** → `ReloadCerts` to the active generation.
  - `keeper.sock`: each client line is forwarded as `Control(line)`, and the `ControlReply` written back. For
    `Status`, the keeper adds `"keeper": { "generation": { "pid", "version", "on_trial" } }`, and `format` prints it
    as a `generation<TAB><pid><TAB><version><TAB>trial|confirmed` line after the server line.
  - `control::request` clients connect to `keeper.sock` (spec: "the path stays the same across handoffs"). The
    generation no longer binds `control.sock`; it answers `Control` messages through the same `ControlBackend`.
  - `generation::run(channel_fd: RawFd) -> Result<(), String>`:
    1. Read `Hello` and refuse a `protocol` it does not speak.
    2. Load the config.
    3. Build `ServeState`. In managed mode, run `managed::start` without the lock, then `apply_pending_group(own
       version)`.
    4. `Serving::start(read_udp = false)`, then `Ready`.
    5. On `StartUdp`, `start_udp`.
    6. On channel EOF (Review Focus 1), stop accepting, wait idle up to 30 s, and exit.
    7. Draining (Task 8) stops auto-update and refuses release commands.
  - `notify::ready()`: a datagram `READY=1` to `$NOTIFY_SOCKET` if set (an abstract socket name when it starts with
    `@`); a no-op otherwise.
  - `binary::ProcessInspector`, implementing `BinaryInspector`. It runs `<binary> manifest` with a 10 s timeout and
    parses stdout.
  - `binary` test hooks, under `#[cfg(feature = "test-hooks")]`:
    - A file `<exe>.test-version` next to the running executable overrides the version it reports (in `--version`,
      `manifest`, `Ready` and the server manifest).
    - A file `<exe>.test-exit-after-ready-ms` makes a generation exit that long after `Ready`.
    - A file `<exe>.test-never-ready` makes it never send `Ready`.
    - These let tests build versions 0.2.0 and 0.3.0 by copying the one test binary.
  - Config: `[handoff] ready_timeout`, `drain_timeout`, `trial_period` (humantime, defaults as above, each at least
    1 s).

- [ ] **Step 1: Write the failing tests** in `tests/keeper.rs`:
  - `keeper_starts_a_generation_and_serves` (`GET /` works; `status` shows a `generation` line with a PID other than
    the keeper's).
  - `crashed_generation_is_restarted`: `kill -9` the generation PID from `status`; within 3 s `GET /` works again and
    `status` shows a new PID; the log has `generation_exited`.
  - `crash_loop_backs_off`: a binary copy with `test-exit-after-ready-ms = 0`; over 4 s at most 4 starts are logged,
    with growing gaps.
  - Review Focus 1, `generation_exits_when_keeper_is_killed`: `kill -9` the keeper; the generation PID is gone within
    3 s.
  - `sigterm_drains_and_exits_zero`.
  - `sighup_reloads_certificates`.
  - `notify_ready_sent`: a test `UnixDatagram` set as `NOTIFY_SOCKET` receives `READY=1` after the first `listening`
    event.
  - Review Focus 5:
    - `stale_keeper_socket_is_replaced` (a dead socket file exists before start → the keeper starts and `status`
      works).
    - `live_keeper_socket_refuses_second_keeper` (the existing second-instance test, now through `keeper.sock`).
  - `immutable_keeper_restarts_crashes_and_writes_nothing` (plan 1's `writes_nothing`, with a crash in the middle).
  - `managed_first_start_registers_binary`: after start, `releases/server/<version>/speedyspec-server` exists and
    `state.json` has `server.confirmed`.
  - `protocol_mismatch_is_refused` (a `Hello` with `protocol: 99`, sent by a test keeper stub, makes the generation
    exit 1 with a message naming 99).
- [ ] **Step 2: Run** `cargo test -p speedyspec-server --features test-hooks --test keeper` → FAIL.
- [ ] **Step 3: Implement.**
- [ ] **Step 4: Run** `cargo test --workspace --features speedyspec-server/test-hooks` → PASS on macOS and Linux musl.
- [ ] **Step 5: Commit.** `feat: keeper process that owns the sockets and supervises generations`

---

### Task 7: Upgrade by socket handoff, the trial and fall-back

**Files:**
- Modify: `crates/speedyspec-server/src/keeper.rs`, `src/generation.rs`, `src/control.rs`
- Test: `crates/speedyspec-server/tests/upgrade.rs`

**Interfaces:**
- Consumes: Tasks 3, 5 and 6.
- Produces: the spec's "Upgrading the server" and "Trial and fall-back", step by step:
  - **Asking for an upgrade.** The control backend, given `Switch` targets that include the server, calls
    `begin_server_switch`, then sends `Upgrade { version, binary }` to the keeper. It replies when the keeper reports
    the result: `switched server <v> (on trial for <trial_period>)`, or the error.
  - **The keeper on `Upgrade`:**
    1. Refuse while a trial or drain is running (Review Focus 3), naming the version on trial.
    2. Start N+1 from `binary` with the same sockets, with `read_udp` false.
    3. Wait for `Ready` up to `ready_timeout`. A timeout, or N+1 exiting first, kills N+1, tells N
       `Resume { failed: Some(v) }`, and fails the request (Review Focus 2).
    4. Send N `Drain`. N runs `Serving::stop_accepting`, which releases UDP, stops auto-update and refuses release
       commands, then sends `UdpReleased`.
    5. Send N+1 `StartUdp`. N+1 is now active, and `keeper.sock` forwards to it.
    6. Start the trial timer.
    7. N waits idle up to `drain_timeout`, but keeps its in-memory state until the trial ends.
  - **N+1 exits before the trial ends:**
    - If N is alive: `Resume { failed: Some(v) }`. N runs `Serving::resume`, then `revert_pending_group`, and becomes
      active again.
    - Otherwise: start a fresh generation from the confirmed binary; its `boot` reverts the group.
    - Either way, log `trial_failed`.
  - **The trial ends:**
    1. `confirm_server(state, v)`, under the state lock.
    2. Send N `Shutdown`, so it finishes draining and exits.
    3. Log `trial_confirmed`.
  - **Rolling the server back** (`release rollback server [v]`) is an upgrade to an older kept binary: the same steps,
    and a group when the current apps require it. Plan 2's rollback pin applies to the server app too.
  - **`SIGTERM` during an upgrade** (Review Focus 4): `Drain` both generations, wait for both (each bounded by
    `drain_timeout`), exit 0.
- `status` shows the trial: `generation<TAB><pid><TAB><version><TAB>trial` until it is confirmed.

- [ ] **Step 1: Write the failing tests** in `tests/upgrade.rs`. Use binary copies with `.test-version` files,
  published through plan 2's `ReleaseHost`, extended with `publish_server(version, binary)` for the test's target
  triple. Use short timeouts: `trial_period = "3s"`, `ready_timeout = "5s"`, `drain_timeout = "5s"`.
  - `upgrade_switches_generation_and_confirms`: install and switch `server@0.2.0`; `status` shows the new PID on
    trial; after 3 s it shows `confirmed`; `state.json` has `server.confirmed = 0.2.0`; the old PID has exited.
  - `failed_trial_resumes_old_generation`: 0.2.0 with `test-exit-after-ready-ms = 1000` → within 5 s `status` shows the
    old PID active again, `server.failed` holds 0.2.0, and `GET /` never failed during the run.
  - `failed_trial_after_old_exited_starts_confirmed_binary`: `drain_timeout = "1s"` and
    `test-exit-after-ready-ms = 2500` → a fresh generation of the confirmed version serves.
  - Review Focus 2:
    - `never_ready_binary_is_refused`.
    - `binary_exiting_before_ready_is_refused`. In both, the old generation is never interrupted and the version is
      failed.
  - Review Focus 3, `second_upgrade_refused_during_trial`.
  - `group_switch_with_server`: `admin@2.0.0` requires `server ^0.2`, and server 0.2.0 requires `admin ^2`. Switching
    either alone is refused; together they succeed, and `GET /` serves `admin-2.0.0` from the new generation.
  - `failed_group_trial_reverts_apps`: as above, with a 0.2.0 that exits during the trial → `GET /` serves
    `admin-1.0.0` again.
  - `server_rollback_is_an_upgrade_to_older_binary`.
  - Review Focus 4, `sigterm_during_upgrade_exits_cleanly` (no PID from `status` is left running).
  - `draining_generation_refuses_release_commands` (a direct `Control` to N during the drain → the refusal message).
- [ ] **Step 2: Run** `cargo test -p speedyspec-server --features test-hooks --test upgrade` → FAIL.
- [ ] **Step 3: Implement.**
- [ ] **Step 4: Run** the whole suite with the feature → PASS on macOS and Linux musl.
- [ ] **Step 5: Commit.** `feat: server upgrades by socket handoff with a trial and automatic fall-back`

---

### Task 8: Handoff under load

**Files:**
- Create: `crates/speedyspec-server/tests/handoff_under_load.rs`
- Test: that file

**Interfaces:**
- Consumes: Task 7, and plan 2's load-test structure (`switch_under_load.rs`, with its recorded HTTP/3 errors and
  server log on failure).

Spec "Testing": "Handoff under load: the same load while the server upgrades; zero failed HTTP requests; an open
WebSocket stays served by the old generation until closed; a WebTransport session receives the upgrading close code."
- WebSocket and WebTransport endpoints arrive with project 4, so this plan tests their transport-level stand-ins (my
  call).
- A slow streaming HTTP/1.1 response (`/__speedyspec_test/slow`, a test-hooks-only route) stands in for a long-lived
  connection that stays on the old generation.
- An open HTTP/3 connection that receives the `0x5353` close stands in for the WebTransport session's close code.

- [ ] **Step 1: Write the failing test** `handoff_under_load`:
  - Load over HTTP/1.1, h2c, HTTPS/h2 and h3 for 10 s, on new and reused connections.
  - Upgrade 0.1.0 → 0.2.0 at 2 s; at 6 s roll back to 0.1.0 by installing 0.3.0 and switching (each confirmed after
    a 1 s trial).
  - Assert:
    - zero failed TCP requests;
    - h3 requests either succeed, or fail only with the `0x5353` close and succeed on immediate retry, with every
      other HTTP/3 error counted as a failure;
    - a slow response opened before the first upgrade completes with all its bytes, from the old generation's PID
      (the route echoes `getpid()`).
- [ ] **Step 2: Run** with the feature → FAIL until the route exists.
- [ ] **Step 3: Add the test-hooks route** in the generation's router (`#[cfg(feature = "test-hooks")]`): `GET
  /__speedyspec_test/slow?seconds=N` streams N chunks, one per second, each holding the PID.
- [ ] **Step 4: Run** on macOS and Linux musl, five times each → PASS every time. If HTTP/3 flakes as in plan 2's
  `switch_under_load`, investigate with its recorded errors before going on, and do not retry blindly.
- [ ] **Step 5: Commit.** `test: zero failed requests across server upgrades under load`

---

### Task 9: `reload` and the keeper re-executing itself

**Files:**
- Modify: `crates/speedyspec-server/src/keeper.rs`, `src/main.rs`, `src/control.rs`
- Test: `crates/speedyspec-server/tests/reload.rs`

**Interfaces:**
- Produces:
  - `speedyspec-server reload [--config]` sends `Reload` (a new control request) to `keeper.sock`.
    - The keeper re-reads the config and compares the listeners (addresses, TLS on or off, `http3`).
    - A difference is refused: `listener changes need a full restart (<what changed>)`.
    - Otherwise it runs Task 7's upgrade with the confirmed binary as N+1, trial included.
    - Immutable mode refuses with plan 2's immutable message.
  - Keeper re-exec (spec "Upgrading the keeper itself"):
    - At the end of a successful trial, if the confirmed version is newer than the keeper's own, the keeper prepares
      to `execve` the confirmed binary as `run --resume`.
    - First it clears close-on-exec on the listening sockets, `keeper.sock`'s listener, `server.lock` and the active
      generation's channel.
    - Then it passes their fd numbers and the active generation's PID, version and protocol in
      `SPEEDYSPEC_KEEPER_STATE` (JSON).
    - The re-executed keeper adopts all of these, does not re-bind or re-spawn, and logs `keeper_reexecuted`.
    - The generation is never interrupted.
  - The server manifest printed by `manifest` now sets `protocol: Some(PROTOCOL)`. A keeper refuses a binary whose
    protocol is newer than its own: `binary <v> needs keeper protocol <p>; this keeper speaks <q>`. This only happens before the keeper's own
    re-exec.

- [ ] **Step 1: Write the failing tests:**
  - `reload_picks_up_new_mount`: add a mount to the config and `reload` → the new path serves, and the PID changed.
  - `reload_refuses_listener_change`.
  - `reload_refused_in_immutable_mode`.
  - `keeper_reexecs_after_upgrade`: upgrade to 0.2.0 → after the trial, the keeper PID is unchanged; `status` shows
    keeper version 0.2.0 (add a `keeper` line to `status`); there is a `keeper_reexecuted` event; `GET /` never failed.
  - `keeper_reexec_keeps_listeners_and_socket` (`keeper.sock` and both ports still answer).
  - `newer_protocol_binary_refused` (a binary copy with `.test-protocol = 99`).
- [ ] **Step 2: Run** → FAIL.
- [ ] **Step 3: Implement.** Clear `FD_CLOEXEC` with `rustix::io::fcntl_setfd`, and call `rustix::runtime::execve`,
  or `std::os::unix::process::CommandExt::exec` with the fds kept open.
- [ ] **Step 4: Run** the whole suite with the feature → PASS on macOS and Linux musl.
- [ ] **Step 5: Commit.** `feat: reload by handoff, and the keeper re-executing itself after an upgrade`

---

### Task 10: Docs, CI and the README for operators

**Files:**
- Modify: `.github/workflows/ci.yml` (`--features speedyspec-server/test-hooks`), `docs/handoff.md`
- Create: `README.md` (none exists yet), `docs/systemd/speedyspec-server.service`
- Test: none new; the full suite is the check

**Interfaces:** none.

- [ ] **Step 1: Write the README.** Cover:
  - what the server is;
  - the two modes;
  - the commands, with sample `status` output;
  - the config keys (listeners, apps, mounts, `[handoff]`, `[release.static]`, `[proxy]`, `[access]`, `[limits]`);
  - managed-mode upgrades and rollbacks, including that a rollback pins;
  - the trial;
  - `reload`;
  - running under systemd and in containers;
  - immutable mode needing an existing `run_dir` for `release list`/`status`.

  Every example must match the code; take sample output from a real test run.
- [ ] **Step 2: Write the systemd unit:** `Type=notify`, `ExecStart=… run --config /etc/speedyspec/server.toml`,
  `ExecReload=… reload`, `KillMode=mixed`, `Restart=on-failure`, `RuntimeDirectory=speedyspec`,
  `StateDirectory=speedyspec`.
- [ ] **Step 3: Update CI and run** the full suite locally on macOS and Linux musl with the feature → PASS.
- [ ] **Step 4: Commit.** `docs: operator README, systemd unit and CI for keeper tests`
