# Keeper and socket handoff

`speedyspec-server` (`keeper.rs`, `generation.rs`, `control.rs`, `serving.rs`, `binary.rs`),
`speedyspec-server-handoff`. From the keeper and socket handoff plan (#2), and two findings recorded 2026-11-21.

## The supervisor spins on a closed events channel

- **Where:** `keeper.rs`, `supervise`, the `current.events.recv()` arm.
- **What happens:** once a generation's channel closes, `recv()` returns `None` at once on every turn of the loop. The
  arm does nothing with it, so the keeper busy-loops at full CPU until `exited()` resolves for that generation.
- **Suggested fix:** stop polling the events of a generation whose channel has closed, for example by taking the
  arm out with a flag or a `select!` guard.

## Keeper re-exec hazards

- **Where:** `keeper.rs`, `Keeper::reexec` and `resume`.
- **What happens:** several small things can go wrong across the keeper's `execve`:
  - `next_id` starts again at 1, so a reply from the adopted generation to an old request id could match a new
    request.
  - `keeper.sock` clients with a request in flight get EOF. `status`, `list` and `check` are retried once; other
    requests report that the server closed the connection.
  - `SPEEDYSPEC_KEEPER_STATE` stays in the keeper's environment after the exec, so every later child (each new
    generation) inherits it.
  - the fd numbers in that variable are trusted without checking that they are open and of the right kind.
  - the resumed keeper loads the config file again before resuming; if the file was broken since start-up, the
    re-exec fails and the keeper exits.
- **Suggested fix:** carry `next_id` in the saved state; remove the variable after reading it; check each fd with
  `fstat` before adopting it; resume from the config the previous keeper used.

## ChildProc::adopt ignores waitpid errors

- **Where:** `keeper.rs`, `ChildProc::adopt`.
- **What happens:** the thread calls `waitpid` once and publishes whatever status it got. If `waitpid` returns -1
  (for example `EINTR`), the keeper believes the adopted generation exited with status 0 while it is still running.
- **Suggested fix:** retry on `EINTR`, and treat other errors as "unknown" rather than as an exit.

## reload ignores certificate paths and [handoff] changes

- **Where:** `keeper.rs`, `Keeper::reload`.
- **What happens:** reload compares only each listener's address, whether it has TLS, and whether HTTP/3 is on. A
  changed certificate or key path, or a changed `[handoff]` section, is accepted without a word, but the keeper keeps
  the sockets and settings it started with.
- **Suggested fix:** compare those too, and refuse the reload (or say a restart is needed) when they changed.

## ConfirmServer can be sent by any keeper.sock client

- **Where:** `keeper.rs` (`KeeperControl` forwards every line), `control.rs` (`ConfirmServer`),
  `speedyspec-server-app` `releases.rs` (`confirm_server_trial`, `confirm_server`).
- **What happens:** `ConfirmServer` is meant for the keeper only, but a client of `keeper.sock` can send it, and it is
  forwarded to the generation. `confirm_server_trial` does not check that the version matches `server.trial`, so a
  client can confirm any version. The free function `confirm_server` is used only by tests.
- **Suggested fix:** refuse keeper-only requests that arrive from `keeper.sock`; check the version against the trial;
  move `confirm_server` behind `cfg(test)` or into the tests.

## run_dir and the control socket's staging name use umask permissions

- **Where:** `control.rs`, `serve`; `keeper.rs`, `hold_server_lock`.
- **What happens:** `run_dir` is created with `create_dir_all`, so outside systemd it is usually 0755. The socket is
  bound as `keeper.sock.<pid>` under the umask and made 0600 before it is renamed to `keeper.sock`; under umask 0002
  the group can connect during that short window. The request line is read with no size limit and no idle timeout
  (low impact, since the socket is owner-only).
- **Suggested fix:** create `run_dir` with mode 0700 (`DirBuilder::mode`), bind the socket under a restrictive umask
  or inside a 0700 directory, and bound the request line and its read time.

## Immutable mode creates server.lock in run_dir

- **Where:** `keeper.rs`, `hold_server_lock`.
- **What happens:** immutable mode is meant to write nothing to disk, and it does not create `run_dir`. But it still
  opens `server.lock` with `create(true)`, so a regular file appears in `run_dir`.
- **Suggested fix:** in immutable mode, lock an existing file only, or lock the socket path itself.

## Edge cases in passing fds to a generation

- **Where:** `keeper.rs`, `Keeper::spawn` (`dup2(child_fd, 3)`); `speedyspec-server-handoff`, `channel.rs`
  (`MAX_FDS`, `recv_once`).
- **What happens:**
  - if the channel's fd is already 3, `dup2(3, 3)` does nothing and leaves close-on-exec set, so the child starts
    without its channel.
  - more than 32 listening sockets fail only when the `Hello` message is sent, not when the config is loaded.
  - `MSG_CTRUNC` is not checked after `recvmsg`, so fds dropped by the kernel go unnoticed.
- **Suggested fix:** clear close-on-exec with `fcntl` when the fd is already 3; refuse more than 32 sockets at config
  load; fail the receive when `MSG_CTRUNC` is set.

## A generation whose keeper is gone ignores signals while it drains

- **Where:** `generation.rs`, the main loop.
- **What happens:** when the keeper's channel closes (EOF), the generation drains for `ORPHAN_DRAIN` without
  listening for SIGTERM, so a stop during that time waits for the drain. If reading the channel fails with an error
  instead of EOF, the generation exits at once without draining.
- **Suggested fix:** keep watching signals during the orphan drain, and treat a channel error like EOF.

## The supervisor runs slow calls inline

- **Where:** `keeper.rs`, `supervise`: `ProcessInspector.manifest` on an upgrade, and the `ConfirmServer` forward at
  the end of a trial.
- **What happens:** both run inside the supervisor's loop. `manifest` is a blocking call (bounded by a 10 s
  timeout) on the async thread. The `ConfirmServer` forward has no timeout at all. While either runs, the keeper
  handles nothing else: no crash restart, no signal, no other command.
- **Suggested fix:** run `manifest` with `spawn_blocking`, and put a timeout on the confirm forward.

## Gaps in the keeper's tests

- **Where:** `crates/speedyspec-server/tests/keeper.rs`, `upgrade.rs`.
- **What happens:** there is no test for SIGTERM while the first generation is starting. The start-up passes a stop
  flag that never fires (`start_ready(&confirmed, &never)`), so such a SIGTERM waits until the generation is ready
  or `ready_timeout` passes. The review also asked for a test of the keeper's exit code; tests cover exit 0 after
  SIGTERM and exit 1 on a protocol mismatch, but not the other exit paths.
- **Suggested fix:** add both tests, and let SIGTERM stop the first start.

## CI never builds the server without test-hooks

- **Where:** `.github/workflows/ci.yml`.
- **What happens:** every CI build and test of `speedyspec-server` turns on `test-hooks`. A build break that only
  shows without the feature is first seen by the release build, which builds the binary without it.
- **Suggested fix:** add a `cargo build -p speedyspec-server` (or `cargo check`) without the feature to the test job.

## Documentation drift

- **Where:** `docs/handoff.md`, `crates/speedyspec-server/src/control.rs`.
- **What happens:**
  - the review found two passages of `docs/handoff.md` that no longer match the code: the "Trial ends" step (the
    keeper asks the new generation to confirm the version in `state.json`; in a cluster, `confirm_server_trial`
    writes the pod's member report instead), and the re-exec step's "after `execve` nothing could reap it". The
    review's exact objection to the second was not recorded.
  - the module doc of `control.rs` still says the socket is `<run_dir>/control.sock`; it is `keeper.sock`.
  - the `control.sock` left by the managed releases plan's servers is never removed.
- **Suggested fix:** correct the wording, and remove a stale `control.sock` at start-up.

## status can be refused while state.json already says confirmed

- **Where:** `keeper.rs`, `run_async` (with `managed.rs`, `confirmed_binary`, which calls `register_running_binary`).
- **What happens:** in managed mode the keeper writes the confirmed version to `state.json` before `control::serve`
  binds `keeper.sock`. During that window, `status` gets "connection refused" though the state on disk already says
  the server is confirmed. The test now waits for the socket (#29); the window itself remains.
- **Suggested fix:** bind `keeper.sock` before registering the running binary, or have `status` retry a refused
  connection briefly when the state file exists.

## HTTP/3 connections can be closed with the wrong code during an upgrade

- **Where:** `serving.rs`, `release` (`CLOSE_FLUSH`).
- **What happens:** `Endpoint::close` only queues a close on each connection's driver task. The code then sleeps a
  fixed 50 ms (`CLOSE_FLUSH`) before moving the endpoint to a throwaway socket and stopping. Under load the close
  packets may not have left by then, so a connection can be closed by the server's stop with another code, not the
  upgrade code `0x5353` that tells clients to reconnect.
- **Suggested fix:** wait for the connections' close to be sent (for example `Endpoint::wait_idle` with a bound)
  instead of a fixed sleep.
