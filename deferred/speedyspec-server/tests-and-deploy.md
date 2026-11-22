# Tests and deployment examples

`speedyspec-server` (tests), `speedyspec-server-infra-fs` (tests), `deploy/kubernetes`. Recorded 2026-11-21.

## Cluster tests upgrade to the literal version "0.2.0"

- **Where:** `crates/speedyspec-server/tests/cluster.rs`: `rolling_upgrade_goes_pod_by_pod`,
  `rolling_upgrade_stops_and_reverts_on_one_failed_trial`, `pod_that_missed_a_rollout_catches_up_on_start`.
- **What happens:** these tests publish and install a server `0.2.0` and expect the pods to move to it. The server is
  now 0.1.1. Once it reaches 0.2.0 itself, the tests would "upgrade" to the version already running and fail or
  prove nothing. The same kind of fixed version broke the release of 0.1.1 (#30).
- **Suggested fix:** derive the target from `CARGO_PKG_VERSION` (for example its next major), as #30 did for the
  revert check.

## torn_member_file_keeps_last_good_copy can see no log lines

- **Where:** `crates/speedyspec-server-infra-fs/tests/cluster.rs`, `torn_member_file_keeps_last_good_copy`.
- **What happens:** the test counts `member_unreadable` lines in captured logs and expects exactly one. It failed once
  on macOS with zero lines when run in parallel with the other tests, and passes alone. The log capture is probably
  shared between tests running at the same time.
- **Suggested fix:** capture logs per thread (a scoped subscriber), or run this test on its own.

## file_lease_view_turns_false_at_renew_deadline_without_a_renewal has a tight wall-clock bound

- **Where:** `crates/speedyspec-server-infra-fs/tests/cluster.rs`,
  `file_lease_view_turns_false_at_renew_deadline_without_a_renewal`.
- **What happens:** the test times how long the leader takes to notice it has stopped renewing, and asserts
  `gone <= renew_deadline + retry`. A slow runner overshoots that. It failed on macOS CI in #30's run.
- **Suggested fix:** keep the bound that matters (`gone < lease`, before the lease could expire), and loosen or drop
  the tight one.

## The Kubernetes example configures only the server and a static admin

- **Where:** `deploy/kubernetes/configmap.yaml`.
- **What happens:** the example config has the server app (from releases.speedyspec.com) and a static `admin` app
  from `releases.example.com`, mounted at `/admin/` as signed-in. It has no PHP runtime app, no admin API (the gate
  app), no blog and no front-end API. Someone starting from it gets a server that cannot run PHP or check sign-ins
  against the admin API.
- **Suggested fix:** add the runtime, the gate app and the PHP apps, with placeholder sources, and a `[gate]` section.
