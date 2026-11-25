# mu-cd

<p align="center">
  <a href="https://hook.lma1980.duckdns.org/jobs"><img src="https://hook.lma1980.duckdns.org/badges/LMA1980/mu-cd/status.svg" alt="CI Status"/></a>
  <a href="https://hook.lma1980.duckdns.org/jobs"><img src="https://hook.lma1980.duckdns.org/badges/LMA1980/mu-cd/score.svg" alt="Quality Score"/></a>
  <a href="https://hook.lma1980.duckdns.org/jobs"><img src="https://hook.lma1980.duckdns.org/badges/LMA1980/mu-cd/coverage.svg" alt="Coverage"/></a>
  <a href="https://hook.lma1980.duckdns.org/jobs"><img src="https://hook.lma1980.duckdns.org/badges/LMA1980/mu-cd/benchmark.svg" alt="Benchmark"/></a>
</p>

______________________________________________________________________

Continuous Delivery (CD) platform for the K3 ecosystem orchestrating zero-downtime, autonomous, verified deployments across cluster worker nodes. `mu-cd` coordinates node leases, stages candidate binaries and static web bundles in symmetric A/B Btrfs subvolumes (`@a` \<-> `@b` via `<service>/current`), executes sandboxed live health and integration gates, verifies canary SLO bake windows, and performs sub-second atomic rollback with automated release tainting.

See the [A/B Btrfs deployment architecture](docs/adr/ADR-051-mu-cd-ab-btrfs-deployment.md).

______________________________________________________________________

## Workspace Submodules

| Submodule                             | Target | License                                       | Responsibility                                                                                             |
| ------------------------------------- | ------ | --------------------------------------------- | ---------------------------------------------------------------------------------------------------------- |
| [`mu-cd-controller`](code/controller) | native | AGPL-3.0-only                                 | Cluster-wide CD orchestrator, lease management, canary SLO verification, `/api/cd/*`, and release tainting |
| [`mu-cd-agent`](code/agent)           | native | AGPL-3.0-only                                 | Host-level daemon managing per-service `@a`/`@b` slots, atomic `current` swap, IT gates, and watchdogs     |
| [`mu-cd-ipc`](code/ipc)               | native | LGPL-3.0-only WITH LGPL-3.0-linking-exception | Framed mTLS IPC protocol definitions, zero-copy serialization (`fory`/`zerocopy`), and lease RPC schemas   |

## Ecosystem Integrations

- **[`mu-ci-controller`](https://github.com/LMA1980/mu-ci-controller)**: Dispatches post-CI rollouts via `POST /api/cd/deploy` after successful `main`-branch builds.
- **[`mu-registry`](https://github.com/LMA1980/mu-registry)**: Hosts CAS release artifacts and integration-test (IT) bundles, receiving canary promote/taint updates.
- **[`mu-observable-hub`](https://github.com/LMA1980/mu-observable-hub)**: Evaluates canary bake window SLO assertions via `MuObserveSloClient` and ingests `MUOB`-framed telemetry emitted via `mu-observable-ipc`.
- **[`mu-ci-isolation`](https://github.com/LMA1980/mu-ci-isolation)**: Provides `KernelSandbox` isolation for preselected CAS integration-test bundle execution inside `mu-cd-agent`.
- **[`mu-auth`](https://github.com/LMA1980/mu-auth)**: Enforces `PolicyFolio` RBAC (including `super-operator` and `ab:rollback`, with zero `ab:promote` endpoint) on interactive `/api/cd/*` routes.
- **[`mu-console`](https://github.com/LMA1980/mu-console)**: Consumes `/api/cd/*` snapshots and SSE streams to visualize active slots, live IT gate progress, and emergency rollbacks.

## Architecture & Lifecycle

```text
[ Idle ]  <── Triggered via POST /api/cd/deploy (mu-ci-controller) or CLI
   │
   ▼
[ Discover Idle Node ] ──(Active Jobs Running)──> [ Exponential Backoff ]
   │
   ▼
[ Acquire TTL Deployment Lease ]
   │
   ▼
[ Snapshot Active Slot (snapshots/@snap-<digest>) & Stage Candidate in Inactive Slot (@b or @a) ]
   │
   ▼
[ Atomic Symlink Swap (<service>/current -> @b or @a) ]
   │
   ▼
[ Watchdog Probe, KernelSandbox IT Bundle Gate & Canary Bake SLO Gate (mu-observable-hub) ]
   ├── All Passed ──> [ Commit Symmetric Active Slot (@a <-> @b) & Promote in mu-registry ] ──> [ Complete ]
   └── Fail / Timeout ──> [ Atomic Rollback (<service>/current -> restore_slot) ] ──> [ Taint in mu-registry ]
```

## Policy Repository Configuration

Continuous delivery deployment specifications are managed in the central CD policy repository (`MU_CD_DEPLOY_REPO_DIR`) and triggered via digest-only CI release events (`POST /api/cd/deploy`), decoupling service repositories from per-repo `.mucd/` or `.muconde/` manifests.

## Building & Verification

```bash
# Pre-PR local verification gate
(cd code && cargo fmt --all -- --check)
./scripts/format_docs.sh --check
(cd code && cargo clippy --workspace --all-targets -- -D warnings)
(cd code && cargo test --workspace)
cargo agents plugin validate Symposium.toml --no-check-crates
```

## License

`mu-cd` (workspace root, controller, and agent) is licensed under the GNU Affero General Public License v3.0 (**AGPL-3.0-only**). See [LICENSE](LICENSE).

In accordance with the K3 platform Two-Tier Licensing Policy:

- IPC protocol crate ([`mu-cd-ipc`](code/ipc)) is licensed under the GNU Lesser General Public License v3.0 with the LGPL-3.0 Linking Exception (**LGPL-3.0-only WITH LGPL-3.0-linking-exception**).
- Autonomous daemons and controllers ([`mu-cd-controller`](code/controller), [`mu-cd-agent`](code/agent)) are licensed under **AGPL-3.0-only**.
