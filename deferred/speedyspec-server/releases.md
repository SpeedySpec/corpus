# Managed releases

`speedyspec-server-app`, `speedyspec-server-infra-fs`, `speedyspec-server-infra-http`,
`speedyspec-server-infra-minisign`, `speedyspec-server`. From the managed releases plan (#1).

## Prune deletes releases before it saves the state

- **Where:** `speedyspec-server-app`, `releases.rs`, `prune`.
- **What happens:** prune removes each old release directory as it goes and saves `state.json` once at the end. If a
  later delete fails, the earlier ones are already gone from disk but the saved state still lists them as kept. The
  call that triggered the prune (a switch, for example) also returns an error, though the switch itself took effect.
- **Suggested fix:** drop the versions from `kept`, save the state, then delete the directories.

## A failed save at boot leaves memory naming the missing version

- **Where:** `speedyspec-server-app`, `releases.rs`, `boot`.
- **What happens:** when the active release is missing and boot falls back to another, it saves the new state. If
  that save fails, the error is logged (`boot_state_not_saved`) but the in-memory state keeps the old, missing
  version. The fallback serves, while `status` and later version comparisons still name the missing one.
- **Suggested fix:** adopt the fallback in memory and report the save error, or refuse to start.

## check_interval = "0s" is accepted

- **Where:** `speedyspec-server`, `config.rs`, where `app.<id>.check_interval` is parsed.
- **What happens:** any duration is accepted, including zero. With `"0s"` the server polls the release index
  without pause.
- **Suggested fix:** refuse a zero interval, or set a sensible minimum.

## Unpacked files are not fsynced before the release is committed

- **Where:** `speedyspec-server-infra-fs`, `unpack.rs` and `store.rs` (`commit`).
- **What happens:** the unpacked files are renamed into place and `state.json` is saved, but the files themselves are
  never synced to disk. After a power loss, a release the state calls kept can have empty or partial files.
- **Suggested fix:** fsync each unpacked file and its directories before the rename.

## A legacy signature is read whole into memory once per configured key

- **Where:** `speedyspec-server-infra-minisign`, `lib.rs`, `MinisignVerifier::check`.
- **What happens:** when streaming verification cannot be used (a legacy, not pre-hashed signature), the whole
  artifact is read into memory. This repeats for every configured key, including keys whose id does not match the
  signature's.
- **Suggested fix:** compare the signature's key id with each key first, and read the artifact only for the matching
  key.

## Artifact size is unbounded before the checksum is checked

- **Where:** `speedyspec-server-infra-http`, `lib.rs`, the download (`std::io::copy` into the file).
- **What happens:** the download writes whatever the source sends. A misconfigured or compromised source can fill the
  disk before the checksum comparison fails.
- **Suggested fix:** an optional `size` in the release index, or a configured maximum, enforced while downloading.

## Duplicate targets in one switch are not refused

- **Where:** `speedyspec-server-app`, `releases.rs`, the switch path.
- **What happens:** `switch admin@1 admin@2` is accepted. Both versions are prepared and activated in turn, and the
  app's `previous` ends up as `1`, not the version that was really active before.
- **Suggested fix:** refuse a switch that names the same app twice.
