# Fusion Tests

## Key Concepts

**Fusion** is Couchbase's disaggregated storage feature: a `magma` bucket's log store is asynchronously uploaded to an external location (NFS or S3), decoupling data durability from local disk. This is distinct from ordinary magma testing in `tests/integration/`.

Fusion is enabled cluster-wide via REST, not couchbase-cli (there is no CLI subcommand for it yet):
1. `POST /settings/fusion` with `logStoreURI` -- must happen **before** any Fusion-backed buckets are created.
2. `POST /fusion/enable` -- turns Fusion on cluster-wide (or for specific buckets).
3. Buckets created afterward with `storage: magma` automatically use Fusion.

Reference implementation: `fusion_sanity.py` / `fusion_base.py` in the TAF repo (`pytests/storage/fusion/`) is the source of truth this test suite is ported from -- `test_fusion_data_lifecycle()` for the full lifecycle, `FusionBase.setUp()` for NFS provisioning, `run_rebalance()` for the accelerated-rebalance flow. `test_fusion_simple.yml` here mirrors the setup + a single rebalance-in, end to end.

## Prerequisites

**1. Must use the `file` provider.** NFS server discovery (`{{.NfsServer}}`, added in commit `6321ea1ad4`) is only implemented for `FileProvider` -- `DockerProvider`, `SwarmProvider`, and `ClusterRunProvider` all `panic()` on `ProvideNFSServer()`. Run with:
```bash
./sequoia -provider file:hosts.json -skip_setup -scope tests/fusion/scope_fusion_simple.yml -test tests/fusion/test_fusion_simple.yml
```

**2. `providers/file/hosts.json` needs an `nfs_server:<IP>` entry**, in addition to the normal node entries. Entries are space-separated, one token per server slot in scope declaration order. Ordinary node entries are **bare IPs** -- do not prefix them (`FileProvider.ProvideCouchbaseServers` in `lib/provider.go` only special-cases the literal `syncgateway:` and `nfs_server:` prefixes; anything else is stored as-is and used directly as the host address, so `local:172.23.100.10` would resolve to the broken host string `"local:172.23.100.10"`):
```yaml
--- "172.23.100.10 172.23.100.11 nfs_server:172.23.100.99"
```
This file is gitignored -- create it locally pointing at real hardware. The NFS server does **not** need to be one of the Couchbase nodes; it's a separate box `setup_nfs_server` provisions from scratch.

**3. Each node needs SSH access with the `ssh_username`/`ssh_password` set in the scope** (`root` / `couchbase` in `scope_fusion_simple.yml`, matching TAF's defaults) -- both for the NFS client mount and for staging/running the Fusion rebalance scripts. The NFS server needs the same credentials.

**4. Requires a Fusion-capable server build** (8.x+) with `/opt/couchbase/bin/fusion/accelerator-cli` present, and Python3 with `venv` available on every node. Match `sequoiatools/couchbase-cli` image tags to the `-version` flag per the root [CLAUDE.md](../../CLAUDE.md) convention.

## What `test_fusion_simple.yml` automates (no manual pre-provisioning needed)

Unlike the earlier draft of this test, NFS server/client setup is no longer an out-of-band manual step -- it's driven by Sequoia itself, mirroring `FusionBase.setUp()` in TAF:

| TAF (`fusion_base.py`) | Sequoia equivalent |
|---|---|
| `setup_nfs_server()` (installs `nfs-kernel-server`, exports `/data/nfs/share`) | `setup_nfs_server` template (`tests/templates/nfs.yml`) |
| `setup_nfs_client()` per node (mounts `/mnt/nfs/share`) | `setup_nfs_client_all_nodes` template |
| `copy_scripts_to_nodes()` per node | `stage_fusion_rebalance_scripts_all_nodes` template |
| `run_rebalance()` | `fusion_rebalance` template |

### Why this needed a Go change, not just YAML

Sequoia's `CompileCommand` (`lib/scope.go`) collapses **all** whitespace in a `command:` string before splitting it into argv -- so a multi-line heredoc (the natural way to push a script's content over SSH) gets mangled; nothing else in this codebase ever sends a multi-line file over SSH this way. Instead, `tests/fusion/scripts/*` are checked into the repo as normal, readable, diffable files, and a new template function -- `{{file_base64 "path/relative/to/repo/root"}}` (`lib/template.go`) -- reads a file at render time and inlines it as a single base64 token (no internal whitespace, survives the collapse, decodes byte-identical on the other end). The templates then do `echo {{file_base64 "..."}} | base64 -d > /path/on/node && chmod +x ...` over one SSH command. This is a generically useful primitive beyond Fusion -- anything needing to push a file to a remote host through Sequoia can reuse it.

`tests/fusion/scripts/` contains verbatim copies from TAF's `scripts/fusion_scripts/`:
- `setup_nfs_server.sh` / `setup_nfs_client.sh` -- the plain (non-rolling-share) variants, chosen because they use fixed paths (`/data/nfs/share` server-side, `/mnt/nfs/share` client-side) with no dynamic output to capture across actions -- Sequoia templates have no clean mechanism to thread a value like TAF's `NEW_SHARE_DIR=` output between actions.
- `run_fusion_rebalance.py`, `run_local_accelerator.sh`, `config.json` -- unmodified.

## `fusion_rebalance`: why it can't be a container, and how it works

`run_fusion_rebalance.py` shells out locally to `/opt/couchbase/bin/fusion/accelerator-cli` and does local file operations against the NFS mount -- both on whatever host it runs on. That means it **must run from an actual Couchbase node** (the `fusion_rebalance` template SSHes to `$0` and runs it there), not from a generic Sequoia driver container the way `sequoiatools/pitr` or `sequoiatools/cbdozer` work. This mirrors TAF running it from the cluster master via `RemoteMachineShellConnection`.

Flow (`stage_fusion_rebalance_scripts_all_nodes` must run first):
1. Build a fresh venv on `$0`, `pip install paramiko requests` (not guaranteed present in the server image's system python).
2. Run `run_fusion_rebalance.py --current-nodes --new-nodes --env local --config /root/fusion/config.json ...`, which: calls `prepareRebalance`, splits/syncs the rebalance manifest, SSHes into every involved node to run its local `run_local_accelerator.sh` (pre-staged there) to pre-copy guest volume data from the log store, then calls `/controller/rebalance`.

`fusion_rebalance` takes explicit node-list args rather than computing them itself (`$0`=driver node, `$1`=current nodes CSV, `$2`=new-topology nodes CSV, `$3`=sleep time, `$4`=rebalance count) -- consistent with how `rebalance_out`/`add_node` in `tests/templates/rebalance.yml` take explicit node args rather than deciding topology themselves.

**Comma-list args must be parenthesized.** Sequoia's `args:` splitter (`ResolveTemplateActions` in `lib/test.go`) breaks on every top-level comma; parens are the escape hatch for a value that legitimately contains one (same convention already used by `.ActiveIndexNode` calls elsewhere in the repo):
```yaml
args: "{{.Orchestrator}}, {{.Orchestrator}}, ({{.Orchestrator}},{{.InActiveNode}}), 60, 1"
#                          ^ $1 current      ^ $2 new (parenthesized, 2 IPs)   ^$3 ^$4
```

**`requires`/`before`/`until` on a call into a `foreach`-bearing template needs `$.DoOnce`, not `.DoOnce`.** `setup_nfs_client_all_nodes` and `stage_fusion_rebalance_scripts_all_nodes` both contain a `foreach`. A `requires:` set on the *call* to one of these gets inherited onto the ranged sub-action (`ResolveTemplateActions`, `lib/test.go:912`) and ends up textually inside the `{{range}}...{{end}}` block the `foreach` expands into. Inside a Go `text/template` range, `.` is rebound to the current loop item (a plain string here), so `.DoOnce` fails with `can't evaluate field DoOnce in type string`. `$` always stays bound to the root context regardless of range nesting, so `$.DoOnce` works in both places. Verified with an isolated `text/template` reproduction before landing the fix -- the failure and the fix are exact, not a guess.

## The bucket is not declared in scope_fusion_simple.yml -- and must not be

`scope_fusion_simple.yml` deliberately has no `buckets:` block. `Test.Run()` runs `scope.SetupServer()` (`lib/scope.go`, node-init through bucket-create) as one atomic Go-code block, entirely *before* any action in `test_fusion_simple.yml` executes -- there is no way to interleave a test action in the middle of it. Since Fusion has to be configured and enabled *before* a Fusion-backed bucket is created (`configure_fusion` → `enable_fusion` → *then* buckets, matching `storage_base.py`), and `CreateBuckets()` is part of that same atomic setup block, declaring the bucket in the scope file would create it before `test_fusion_simple.yml`'s `configure_fusion`/`enable_fusion` ever run.

So the `default` bucket is created explicitly, mid-test, in the `create_bucket` section right after `wait_for_fusion_status`. A consequence: `{{.Bucket}}` (which reads the scope's declared bucket list) has nothing to resolve to here -- every reference uses the literal bucket name `default` instead. If you add a second bucket, declare and create it the same way (an explicit `bucket-create` action after Fusion is enabled), not in the scope file.

## Key Templates Reference

`tests/templates/nfs.yml`:

| Template | Purpose |
|----------|---------|
| `setup_nfs_server` | Install `nfs-kernel-server` and export `/data/nfs/share` on `{{.NfsServer}}` |
| `setup_nfs_client` | Mount the share at `/mnt/nfs/share` on node `$0` |
| `setup_nfs_client_all_nodes` | `setup_nfs_client` looped over every node in the scope |

`tests/templates/fusion.yml`:

| Template | Purpose |
|----------|---------|
| `configure_fusion` | `POST /settings/fusion` -- set `logStoreURI` (call once, before bucket creation) |
| `enable_fusion` | `POST /fusion/enable` -- turn Fusion on cluster-wide or per-bucket |
| `disable_fusion` | `POST /fusion/disable` |
| `wait_for_fusion_status` | Poll `/fusion/status` until `state` matches the given value |
| `get_fusion_status` | One-shot `GET /fusion/status`, for logging |
| `get_active_guest_volumes` | `GET /fusion/activeGuestVolumes` -- non-empty while extent migration is still draining after a rebalance |
| `stage_fusion_rebalance_scripts` | Push the 3 rebalance scripts onto node `$0` at `/root/fusion/` |
| `stage_fusion_rebalance_scripts_all_nodes` | `stage_fusion_rebalance_scripts` looped over every node |
| `fusion_rebalance` | Run the accelerated rebalance from node `$0` |
| `memcached_kill_during_sync` | Force `sync_log_store`, then immediately kill memcached on node `$0` (its own supervisor restarts it) -- validates the log store stays consistent despite the uploading node dying mid-sync. Distinct from the mid-rebalance caution below: a periodic flush retries on its own next cycle, unlike an in-flight rebalance DCP stream. |

## `local://` vs `s3://` log stores

`logStoreURI` uses a scheme prefix: `local:///mnt/nfs/share/buckets` for an NFS-backed local mount, `s3://bucket/path` for S3. This test suite only covers the NFS/`local://` path.

## Extending this test

- **Chaos**: kill `memcached` mid-rebalance via `kill_process` (`tests/templates/rebalance.yml`) to exercise Fusion's uploader recovery, matching `crash_during_test` in TAF.
- **Item-count validation**: verify counts survive rebalance via `cbq`/`attack_query` (`tests/templates/n1ql.yml`).
- **Swap / rebalance-out**: `fusion_rebalance`'s `$1`/`$2` args already support any topology change -- pass a `$2` that drops a node from `$1`'s list for a rebalance-out, or a same-size but different-membership list for a swap.
- **S3 backend**: would need a new `configure_fusion` call with an `s3://` URI and the `"aws"` section of `config.json`; NFS-specific staging (`setup_nfs_*`) wouldn't apply.
