# mu-cd

<p align="center">
  <a href="https://hook.lma1980.duckdns.org/jobs"><img src="https://hook.lma1980.duckdns.org/badges/LMA1980/mu-cd/status.svg" alt="CI Status"/></a>
  <a href="https://hook.lma1980.duckdns.org/jobs"><img src="https://hook.lma1980.duckdns.org/badges/LMA1980/mu-cd/score.svg" alt="Quality Score"/></a>
  <a href="https://hook.lma1980.duckdns.org/jobs"><img src="https://hook.lma1980.duckdns.org/badges/LMA1980/mu-cd/coverage.svg" alt="Coverage"/></a>
  <a href="https://hook.lma1980.duckdns.org/jobs"><img src="https://hook.lma1980.duckdns.org/badges/LMA1980/mu-cd/benchmark.svg" alt="Benchmark"/></a>
</p>

______________________________________________________________________

Continuous Delivery (CD) platform for the K3 ecosystem orchestrating zero-downtime, autonomous, verified deployments across cluster worker nodes. `mu-cd` coordinates node leases, stages candidate binaries in dedicated Btrfs subvolumes, executes live health and integration gates, and provides sub-second atomic rollback with automated release tainting.

See the [A/B Btrfs deployment architecture](docs/adr/ADR-051-mu-cd-ab-btrfs-deployment.md).

______________________________________________________________________

## Workspace Submodules

| Submodule                             | Target | License                                       | Responsibility                                                                                           |
| ------------------------------------- | ------ | --------------------------------------------- | -------------------------------------------------------------------------------------------------------- |
| [`mu-cd-controller`](code/controller) | native | AGPL-3.0-only                                 | Cluster-wide CD orchestrator, lease management, canary SLO verification, and release tainting            |
| [`mu-cd-agent`](code/agent)           | native | AGPL-3.0-only                                 | Host-level daemon managing Btrfs `@a`/`@b` slots, atomic symlink swap, systemd restarts, and watchdogs   |
| [`mu-cd-ipc`](code/ipc)               | native | LGPL-3.0-only WITH LGPL-3.0-linking-exception | Framed mTLS IPC protocol definitions, zero-copy serialization (`fory`/`zerocopy`), and lease RPC schemas |

## Architecture & Lifecycle

```text
[ Idle ]
   │
   ▼
[ Discover Idle Node ] ──(Active Jobs Running)──> [ Exponential Backoff ]
   │
   ▼
[ Acquire TTL Deployment Lease ]
   │
   ▼
[ Stage Candidate Release (Slot B) ]
   │
   ▼
[ Atomic Symlink Swap to Slot B ]
   │
   ▼
[ Watchdog Probe & Live Integration Test Gate ]
   ├── All Tests Pass ──> [ Promote Slot B ➔ Slot A ] ──> [ Release Lease ] ──> [ Complete ]
   └── Failure / Timeout ──> [ Atomic Rollback to Slot A ] ──> [ Taint Artifact in mu-registry ]
```

## Three-Phase Configuration Migration

Continuous delivery deployment specifications adhere to the platform's 3-phase configuration migration policy:

- **Phase 1 (Dual-Stack)**: Side-by-side support for `.mucd/mu-cd.yml` with automatic fallback to legacy `.muconde/MuConDe.yml`.
- **Phase 2 (Cluster Verification)**: Deploy upgraded daemons across production hosts (`node-a`, `node-b`, `worker-riscv64`) and migrate repository manifests to `.mucd/mu-cd.yml`.
- **Phase 3 (Decommissioning)**: Remove legacy `.muconde/` fallback code after all services and repositories have migrated.

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
