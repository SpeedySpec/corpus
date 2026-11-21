# SpeedySpec CLI — Maintaining Kubernetes Installs Implementation Plan (plan 8)

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Everything a zip install can do after `app install` works on a Kubernetes install too:
- setup;
- `app db`;
- `plugin` and `theme`;
- `app backup` and `app restore`.

The database connection gains TLS, for both drivers.

**Architecture:** A *runner pod* does all the work inside the cluster. It is a Job in the install's namespace, scheduled on the node of the app's pod, with the app container's volumes mounted at the same paths. Its one container is a small pinned image that waits (`sleep`). The CLI streams everything through the Kubernetes API's exec:
- this release's own static Linux binary, checked against `SHA256SUMS` (or `--runner-binary`), checked again with `sha256sum` in the pod;
- request files and plugin task files;
- archives and dumps.

It then runs that binary in the pod under hidden `self` modes that reuse the zip code paths unchanged: `run-task`, the `app db` operations, the file swaps, backup and restore. The app's plugin says which workload serves the app and where its root is (a new `Workload` on `AppRuntime`). The CLI knows nothing about WordPress. A maintenance file the plugin names (`.maintenance` for WordPress) is written during plugin and theme updates and restores, on both drivers.

**Tech Stack:** Rust 1.99; `kube` 5 with `ws` (exec) added to `client`, `rustls-tls` and `ring`; `k8s-openapi` 0.29; the `mysql` crate's rustls TLS; the plan 3 FakeCluster for API calls; k3s through `scripts/k3s.sh` for the end-to-end tests; `scripts/mysql.sh` for the database tests.

**Spec:** `docs/superpowers/specs/2026-11-18-speedyspec-cli-app-maintenance-design.md`. Its sections "Running the app's code", "The database" and "Details settled after review" bind it. Its Amendment overrides the rest: the CLI never runs WordPress, and on Kubernetes everything is as designed minus activation, with maintenance mode as file work. This plan is item 4 of the design's "Order of work".

## Global Constraints

- Repositories and branches:
  - `packages/speedyspec-cli`: branch `feat/kubernetes-maintenance` from `origin/main`.
  - `packages/speedyspec-cli-contracts`: branch `feat/workload-contracts` from `origin/main`.
  - `packages/speedyspec-cli-wordpress`: branch `feat/kubernetes` from `origin/main`.
- **Nothing outward-facing without Jacob's go-ahead:** no push, PR, tag, release or workflow run.
- **Stay in this session's lane:** never edit speedyspec-server, the crates copied from it, or the existing PHP area wiring.
- **Never touch a real cluster.** API tests use the FakeCluster. Exec and end-to-end tests use only the throwaway k3s from `scripts/k3s.sh`, through `SPEEDYSPEC_E2E_KUBECONFIG`. They are `#[ignore]`, and fail when it is unset. The token is never printed.
- **Never touch a real database.** Same rules as plans 5 to 7, through `scripts/mysql.sh`.
- **Secrets:** the database password and the cluster token are never in a Job or pod spec, an argument, an environment variable, a log line, an install record or an error message. A secret reaches the runner pod only inside a request file streamed through exec into an `emptyDir`, created 0600 and deleted when the step ends.
- **A runner pod never outlives its command:** it is deleted on success, on failure and on Ctrl-C (the runner's `SignalGuard` pattern), and given `activeDeadlineSeconds` and `ttlSecondsAfterFinished` so a CLI killed outright still leaves nothing for long.
- **The runner's binary is checked twice:**
  1. The CLI checks the release asset for the node's architecture (`nodeInfo.architecture`: `amd64` or `arm64`) against that release's `SHA256SUMS`, with the self-update code. With `--runner-binary`, it computes the sha256 itself.
  2. The pod checks the sha256 again with `sha256sum` before running anything.
- **The CLI never executes WordPress**, on any driver.
- Contracts changes are additive (minor release `1.5.0`); existing classes keep their fields, and new ones have defaults.
- Stage explicit paths, Conventional Commits, `Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>`.
- Before every commit:
  - clippy with `-D warnings`, with its exit code checked;
  - `rustfmt --check` on the touched files.
- The Linux suite at the end, then remove the Docker target volume it built into.

## Review Focus

Each line has a test in the task named in brackets.

1. **A runner pod left behind.** A failing step, a `sleep` image that never pulls, Ctrl-C mid-stream, the node gone, or the CLI killed outright. Each ends with the Job deleted (or, when killed, with the Job's own deadline and TTL), and the error names the stuck phase. [Task 5]
2. **The volumes and the node.**
   - A ReadWriteOnce claim works because the pod runs on the app pod's node.
   - The app container named by `Workload.container` (or the only one) gives the mounts.
   - A workload with no ready pod is refused with a clear message, before any Job is created.
   - `subPath` mounts are copied as they are.
   [Task 5]
3. **Secrets in the cluster.** After every Kubernetes command, the Job spec, the pod spec, `kubectl logs` of the runner and the CLI's output hold neither the database password nor the token. The request file is gone from the `emptyDir`. [Task 6, Task 7]
4. **Big streams.** A 50 000-row dump and a 20 MB upload go through exec in flat memory, both ways. A broken stream mid-way is an error that names the step, never a truncated success. [Task 7, Task 9]
5. **TLS that is not what it claims.** With `tls: verify`, a server whose certificate does not match the CA or host is refused. `tls: required` refuses a server without TLS. The "not encrypted" warning appears only when TLS is off. [Task 2]

## File Structure

```
packages/speedyspec-cli-contracts/
  src/Workload.php, src/MaintenanceFile.php                new
  src/AppRuntime.php (+ ?Workload $workload, ?MaintenanceFile $maintenance)
  src/DatabaseConnection.php (+ string $tls = 'off': off | required | verify, + ?string $caFile)
packages/speedyspec-cli/
  crates/speedyspec-driver/src/app.rs                      mirrors for the new fields
  crates/speedyspec-db/src/lib.rs                          TLS through rustls
  crates/speedyspec-driver-kubernetes/src/exec.rs          new: exec with stdin and stdout streams
  crates/speedyspec-driver-kubernetes/src/runner.rs        new: the runner pod (Job, node, volumes, binary)
  crates/speedyspec-cli/src/app_target.rs                  a Target is a zip root or a Kubernetes workload
  crates/speedyspec-cli/src/remote.rs                      new: the CLI side of each in-pod operation
  crates/speedyspec-cli/src/self_cmd.rs                    hidden in-pod modes
  crates/speedyspec-cli/src/{setup.rs,db_cmd.rs,extensions.rs,backup_cmd.rs}   use the Target's way of reaching files and the database
  crates/speedyspec-cli/tests/kubernetes_maintenance.rs    new (FakeCluster, plus #[ignore] k3s end-to-end tests)
packages/speedyspec-cli-wordpress/
  plugin.php                                               + AppRuntime(workload, maintenance) for Kubernetes installs
```

---

### Task 1: Contracts 1.5.0 and their mirrors

**Interfaces (PHP, `final readonly`):**
- `Workload(string $kind, string $name, string $root, ?string $container = null)`:
  - `kind` is `Deployment` or `StatefulSet`;
  - `root` is the install root inside the container (`/var/www/html`);
  - `container` names the app container when the pod has several.
- `MaintenanceFile(string $path, string $contents)`: `path` is relative to the root. It is written while files are swapped and removed afterwards, even on failure.
- `AppRuntime` gains `?Workload $workload = null` and `?MaintenanceFile $maintenance = null`.
- `DatabaseConnection` gains `string $tls = 'off'` (`off`, `required` or `verify`) and `?string $caFile = null`, a path on the CLI's machine for `verify`.

**Interfaces (Rust):** mirrors in `speedyspec_driver::app`, with checks:
- the kind is one of the two;
- `root` is absolute and has no `..`;
- `container` is a DNS-1123 label;
- the maintenance `path` passes `destination_problem`;
- `tls` is one of the three;
- `caFile` is absolute, and required when `tls` is `verify`.

Objects from 1.4 without these properties decode with their defaults, as `keepUrls` does.

- [ ] Pest tests first, then the classes. Then the failing Rust decode tests, then the mirrors.
- [ ] Commits: `feat: add the workload, maintenance and TLS contracts` and `feat(driver): mirror …`.

### Task 2: TLS for database connections

**Interfaces:** `Connection::open` maps `tls` to the `mysql` crate's `SslOpts`, over rustls:
- `required` encrypts without checking the certificate;
- `verify` checks it against `caFile` and the host name.

The warning in `db_cmd` and `backup_cmd` appears only when `tls` is `off` and the host is not this machine. `scripts/mysql.sh` exposes the MySQL container's CA (`docker cp` of its auto-generated `ca.pem`) as `SPEEDYSPEC_E2E_MYSQL_CA`.

- [ ] **Failing `#[ignore]` database tests:**
  - `required_tls_connects_encrypted` (checks `Ssl_cipher`);
  - `verify_with_the_wrong_ca_is_refused` (a fresh self-signed CA);
  - `verify_with_the_right_ca_connects`;
  - `required_tls_to_a_server_without_it_is_refused` (MariaDB started with TLS off);
  - the warning shows only without TLS.
- [ ] Implement, run on both servers, and commit: `feat(db): encrypt database connections with TLS when the app says so`.

### Task 3: Exec through the Kubernetes API

**Interfaces (`speedyspec-driver-kubernetes::exec`):**
- `Connection::exec(&self, namespace, pod, container, command: &[&str], stdin: Option<&mut dyn Read>, stdout: &mut dyn Write, stderr: &mut dyn Write) -> Result<i32, ClusterError>`.
- It streams both ways in 64 KiB chunks, and gives back the command's exit status.
- A dropped stream is `ClusterError::Other("the stream to <pod> broke while …")`.
- It uses `kube`'s `ws` feature, so add `ws` to the crate's `kube` features.

- [ ] **Failing `#[ignore]` k3s tests:**
  - `exec_streams_both_ways` (a 20 MB stdin echoed through `cat`, compared by sha256);
  - `exec_gives_the_exit_status`;
  - `a_broken_stream_is_an_error` (the pod deleted mid-stream).
- [ ] Implement, and commit: `feat(kubernetes): run commands in a pod with streamed input and output`.

### Task 4: A Target is a zip root or a Kubernetes workload

**Interfaces:**
- `Target::find(name, kube: Option<&ConnectFlags>)`:
  - without cluster flags, as today (the local record);
  - with them, the install record from the cluster (`find_install`), with `Install.root = None` and `driver: "kubernetes"`.
- Every app command (`app db`, `plugin`, `theme`, `app backup`, `app restore`) accepts the Kubernetes driver's connection flags (`--namespace`, `--kubeconfig`, `--context`, `--server`), as `app uninstall` does.
- `Target::workload(runtime) -> Result<Workload, String>` asks `speedyspec_app_runtime`. A Kubernetes install whose plugin gives no `Workload` is refused: "<app>'s plugin does not say which workload serves it on Kubernetes".

- [ ] **Failing FakeCluster tests:**
  - a Kubernetes install is found by name and namespace;
  - one that does not exist is named;
  - the flags of another driver are refused;
  - the missing-workload message.
- [ ] Implement, and commit: `feat: find Kubernetes installs for the maintenance commands`.

### Task 5: The runner pod

**Interfaces (`speedyspec-driver-kubernetes::runner`):** `RunnerPod::start(connection, namespace, workload, binary: RunnerBinary, deadline: Duration) -> Result<RunnerPod, RunnerError>`.
1. Find a ready pod of the workload (its selector) and its node.
2. Copy the named container's `volumeMounts` and the pod's `volumes`.
3. Create a Job:
   - `nodeName` is that node;
   - one container from `--runner-image` (default a pinned `busybox` digest) running `sleep <deadline>`;
   - the mounts are copied, plus an `emptyDir` at `/speedyspec` (medium Memory, 0700);
   - `activeDeadlineSeconds` is the deadline, `ttlSecondsAfterFinished: 60`, and `backoffLimit: 0`;
   - the labels are `speedyspec.runner=1` and the install's name;
   - no environment.
4. Wait for Running. An image that does not pull is named.
5. Stream the binary to `/speedyspec/speedyspec`, `chmod 0700`, and check `sha256sum` against the expected digest.

`RunnerPod::run(args, stdin, stdout, stderr) -> Result<i32, RunnerError>` runs `/speedyspec/speedyspec <args>` through exec. `RunnerPod::put(path, bytes_or_reader, mode)` and `RunnerPod::get(path, writer)` move files. `Drop` deletes the Job (foreground propagation). A `SignalGuard` deletes it on Ctrl-C too.

`RunnerBinary`:
- `Release { version }` downloads this version's asset for the node's architecture and checks it against `SHA256SUMS`, with `speedyspec-selfupdate`'s code; `SPEEDYSPEC_GITHUB_API` lets tests point it at a fake;
- `Local(path)` is `--runner-binary`.

- [ ] **Failing tests:**
  - FakeCluster:
    - the Job spec (node, volumes, mounts, `emptyDir`, deadline, TTL, labels, no env);
    - no ready pod is refused before any Job;
    - a release checksum mismatch is refused before any Job;
    - the Job is deleted when a step fails.
  - `#[ignore]` k3s:
    - `a_runner_pod_sees_the_app_volume` (a file written by the app pod is read by the runner);
    - `the_binary_is_checked_in_the_pod` (a tampered upload is refused);
    - `ctrl_c_deletes_the_job`;
    - `a_killed_cli_leaves_a_job_that_expires`.
- [ ] Implement, and commit: `feat(kubernetes): a runner pod beside the app, with the app's volumes`.

### Task 6: App tasks and setup on Kubernetes

**Interfaces:**
- `runner::run` (the parent side) gains a Kubernetes route:
  1. Stream the plugin's directory (the task file's) to `/speedyspec/plugin/`, and rewrite the task's file path.
  2. Stream the request JSON (0600).
  3. Run `self run-task --request … --result …`.
  4. Stream the result back.
  5. Delete both files.
- The diagnostics and the timeout behave as for zip.
- Setup runs after `wait_and_finish` succeeds, for Kubernetes installs. Plan 6's ruling, which skipped it, is lifted. Its progress is kept in the cluster record: `status: files` and `setup_done`, in a new optional record field, so a re-run resumes.
- `speedyspec_app_database` tasks run the same way, so WordPress's `read-config.php` reads the `wp-config.php` in the volume.

- [ ] **Failing tests:**
  - FakeCluster: the setup progress is recorded and resumed.
  - `#[ignore]` k3s with a fixture app (an nginx Deployment with a volume):
    - `setup_tasks_run_in_the_runner_pod`;
    - `a_failed_task_resumes_on_re_run`;
    - `the_request_file_never_stays` (and Review Focus 3).
- [ ] Implement, and commit: `feat: run setup and app tasks for Kubernetes installs in the runner pod`.

### Task 7: `app db` on Kubernetes

**Interfaces:**
- A hidden `self db --request <file> <export|import|query|tables|size|check> [<sql>]` runs `speedyspec_db` in the pod, with the connection from the request file (deleted at once after reading).
- Export and query stream stdout back. Import streams the dump in through stdin.
- The CLI side is the existing `db_cmd`, with the Target deciding where it runs.
- The connection's host is resolved inside the cluster, so `mysql.<ns>.svc` works.

- [ ] **Failing `#[ignore]` k3s tests** with a MySQL pod in the namespace:
  - `export_and_import_round_trip_through_the_cluster` (CHECKSUM equal);
  - `a_50000_row_export_streams` (CLI peak memory measured);
  - `query_prints_rows`;
  - the password nowhere (Review Focus 3).
- [ ] Implement, and commit: `feat: app db for Kubernetes installs, run inside the cluster`.

### Task 8: `plugin` and `theme` on Kubernetes, and the maintenance file

**Interfaces:**
- **Listing:** a hidden `self shelf-index --root <root> --dir <dir>` writes a tar of each entry's top-level files up to 256 KiB each, enough for headers. The CLI unpacks it into a temp mirror and calls `inspect` on the mirror's paths. Slugs and errors are as for zip.
- **Install and update:**
  - the CLI resolves and downloads the archive, and unpacks it locally with the checked extraction;
  - it streams a tar of the top directory to the pod;
  - a hidden `self shelf-swap --root … --dir … --slug … [--delete]` does the same rename-swap as `extensions.rs`, sharing its code.
- **The maintenance file:** when the app's `AppRuntime` has a `MaintenanceFile`, `update`, `update --all` and `restore` write it before swapping and remove it after, on success or failure, on both drivers. `install` and `delete` do not.

- [ ] **Failing tests:**
  - zip: `the_maintenance_file_is_there_during_an_update_and_gone_after` (the fixture's swap hook checks it exists mid-swap) and `it_is_removed_after_a_failed_update`;
  - `#[ignore]` k3s: install, list, update and delete a plugin on a Kubernetes fixture install.
- [ ] Implement, and commit: `feat: plugin and theme commands for Kubernetes installs, and the maintenance file`.

### Task 9: `app backup` and `app restore` on Kubernetes

**Interfaces:**
- Hidden `self backup-local --request … --to /speedyspec/b.tar.gz` and `self restore-local --request … --archive …` run `flow::backup` and `flow::restore` in the pod. The request carries the `BackupSpec`, the manifest fields, the connection and the move.
- The archive streams out to the CLI's `--to`, or in from the CLI's file.
- The confirmation, the pre-restore backup (written on the CLI's machine) and the messages are as for zip.
- A backup is driver-independent: a zip backup restores onto Kubernetes, and the other way round.

- [ ] **Failing `#[ignore]` k3s tests:**
  - `backup_streams_out_and_restores_in` (same site);
  - `a_zip_backup_restores_onto_kubernetes_with_a_move`;
  - `a_kubernetes_backup_restores_onto_a_zip_install`;
  - big data in flat memory (Review Focus 4).
- [ ] Implement, and commit: `feat: back up and restore Kubernetes installs through the runner pod`.

### Task 10: The WordPress plugin on Kubernetes, end to end, docs

**Interfaces:** for `driver === 'kubernetes'`, the WordPress plugin's `speedyspec_app_runtime` returns `AppRuntime(workload: new Workload('Deployment', <release name>, '/var/www/html', 'wordpress'), maintenance: new MaintenanceFile('.maintenance', '<?php $upgrading = time();'))`. The workload name comes from the install request. On zip, it returns the maintenance file only.

- [ ] `scripts/e2e-kubernetes.sh` (manual, k3s plus the official `wordpress` image): install, setup, `app db`, plugins, backup, and restore onto a zip install and back.
- [ ] CLI README: the Kubernetes maintenance commands, the runner pod and its image, `--runner-binary`, TLS.
- [ ] Run everything, then the Linux suite, then cleanup.
- [ ] Commit each repository.
