# ADR-051: mu-cd Zero-Downtime A/B Btrfs Continuous Delivery & Rollback Lifecycle

**Status**: Accepted

## Context

Self-hosted infrastructure daemons (`mu-ci-worker`, `mu-ci-controller`, `mu-registry`, `mu-cd-agent`, `jero-runner`) running on `node-a`, `node-b`, and `worker-riscv64` require continuous delivery after passing CI verification and publishing release artifacts to `mu-registry`. Naive in-place binary replacement creates three operational hazards:

1. **In-Flight Job Termination & `ETXTBSY`**: Overwriting or restarting a worker daemon while an active compilation or SpacemiT K3 benchmark stage is executing corrupts build state or fails with `ETXTBSY`.
1. **Bricked Host Root Filesystem Risk**: Full-OS A/B rootfs switching (`/` on flash/eMMC on `worker-riscv64`) risks rendering a headless node unbootable and violates NVMe storage boundaries.
1. **Unverified Candidate Promotion**: Promoting a candidate binary without post-swap watchdog health checks and live integration test verification risks stranding a node with a broken control-plane or worker daemon.

## Decision

We adopt `mu-cd` (**Mu Continuous Delivery**), implementing a **Per-Service A/B Btrfs Subvolume Lifecycle** with idle-slot lease gating, atomic symlink switching, post-swap watchdog + integration test verification, and automatic rollback with `mu-registry` artifact SHA-256 tainting.

______________________________________________________________________

### 1. Per-Service Btrfs Subvolume Topology (No Full-OS Rootfs Switching)

On `worker-riscv64`, the dedicated NVMe Btrfs pool mounts dedicated top-level subvolumes directly at their canonical paths without intermediate mounts, provisioning only subvolumes and slots required for services running on that node:

- `@k3-store` → `/var/lib/k3-store`
- `@repository` → `/home/runner/repository`
- `@ollama` → `/usr/local/lib/ollama/.ollama`
- `@mu-ci` → `/var/lib/action-runner`
- `@mucd` → `/srv/mucd` (with legacy fallback `/srv/muconde`)

Within `/srv/mucd` (`@mucd`), `mu-cd-agent` manages strictly scoped nested per-service Btrfs subvolumes under `/srv/mucd/<service>`. Full-OS rootfs switching is **strictly forbidden**; only allowlisted platform services (`ALLOWED_SERVICES = ["mu-ci-worker", "mu-ci-controller", "mu-registry", "mu-cd-agent", "jero-runner"]`, alongside backwards-compatible legacy service identifiers) are managed:

```text
/srv/mucd/<service>/
├── @a/                          # Slot A Btrfs subvolume (Active production binary & state)
│   └── bin/<service>            # Installed with 0755 permissions (`install -m 755`)
├── @b/                          # Slot B Btrfs subvolume (Staging / candidate binary & state)
│   └── bin/<service>            # Installed with 0755 permissions (`install -m 755`)
├── snapshots/                   # Immutable read-only Btrfs rollback snapshots
│   └── @snap-<prev_sha256>/     # Created via `btrfs subvolume snapshot -r @a snapshots/@snap-<sha256>`
└── current -> @a | @b           # Atomic symlink flipped via `ln -sfn` + `rename`
```

Systemd unit files (`<service>.service`) execute `/srv/mucd/<service>/current/bin/<service>`, decoupling the unit definition from the underlying active Btrfs slot.

______________________________________________________________________

### 2. Six-Stage Zero-Downtime A/B Deployment State Machine

Every deployment orchestrated by `mu-cd-controller` (`cd.ci.example.internal:8445`) follows a deterministic 6-stage state machine governed by `.mucd/mu-cd.yml` (or backward-compatible `.muconde/MuConDe.yml`):

```yaml
version: "1"
service: "jero-runner"
target_nodes:
  - "worker-riscv64"
lease_ttl_seconds: 300
warmup_timeout_seconds: 30
integration_tests:
  - "it_health_readiness"
  - "it_k3_rvv_smoke"
```

#### Stage 1: Artifact Pull & Digest Verification

- `mu-cd-controller` receives a release notification from `mu-registry` (`registry.ci.example.internal:8444`) and verifies that the candidate artifact `artifact_sha256` is not marked `tainted: true`.
- Target `mu-cd-agent` nodes download the candidate binary from `mu-registry` over mTLS (`rustls`) and verify its SHA-256 digest before staging.

#### Stage 2: Btrfs Snapshot & Candidate Staging (`prepare_staging_slot`)

- `mu-cd-agent` takes a read-only Btrfs snapshot of the currently active slot (`@a`) into `/srv/mucd/<service>/snapshots/@snap-<prev_digest>`:
  ```bash
  btrfs subvolume snapshot -r /srv/mucd/<service>/@a /srv/mucd/<service>/snapshots/@snap-<prev_digest>
  ```
- `mu-cd-agent` resets staging subvolume `@b` (`btrfs subvolume delete` + `btrfs subvolume create`) and installs the verified candidate binary atomically (`install -m 755 <candidate> /srv/mucd/<service>/@b/bin/<service>`).

#### Stage 3: Idle Worker Detection & TTL-Bounded Lease Acquisition

- Before flipping `/srv/mucd/<service>/current`, `mu-cd-controller` requests a deployment lease (`AcquireLeaseRequest`) from the target node.
- The node grants the lease **only when** `active_slots == 0` (no active CI stage or model inference pass is running on the node). During the lease (`lease_ttl_seconds`, default `300s`), the node is cordoned from new job dispatches.
- If `active_slots > 0`, `mu-cd-controller` waits patiently with exponential backoff until the active stage completes or the lease request times out.

#### Stage 4: Atomic Symlink Swap & Service Restart (`SwapSlotRequest`)

- Once the lease is held and `active_slots == 0`, `mu-cd-agent` atomically updates `/srv/mucd/<service>/current` from `@a` to `@b` (creating a temporary symlink `.current.tmp` and calling POSIX `rename()`) and restarts `<service>.service`.

#### Stage 5: Post-Swap Watchdog Health Probe & Integration Test Gate

- **Watchdog Readiness Window (`warmup_timeout_seconds`, default `30s`)**: `mu-cd-agent` monitors systemd unit state (`ActiveState=active`, zero crash loops) and `/healthz` mTLS readiness across the warmup window.
- **Live Integration Test Execution (`integration_tests`)**: Once healthy, `mu-cd-controller` triggers the integration test suite specified in `.mucd/mu-cd.yml` (`"all"` or explicit test targets `["it_..."]`) against the live Slot B instance.

#### Stage 6A: Promotion to Slot A (On Verification Success)

- When both the watchdog probe and all required `integration_tests` pass:
  1. `mu-cd-agent` synchronizes the verified candidate state into `@a` (`@b` promoted as the canonical production baseline), flips `/srv/mucd/<service>/current -> @a`, and releases the deployment lease.
  1. `mu-cd-controller` records the deployment as `Promoted` in its audit log.

#### Stage 6B: Instant Atomic Rollback & `mu-registry` SHA-256 Tainting (On Failure or Timeout)

- If the process exits unexpectedly during warmup, `/healthz` fails, any integration test fails, or `lease_ttl_seconds` expires:
  1. **Instant Atomic Rollback (`RollbackSlotRequest`)**: `mu-cd-agent` atomically flips `/srv/mucd/<service>/current` back to `@a` (restoring from `snapshots/@snap-<prev_digest>` if `@a` was modified), restarts `<service>.service`, verifies previous-version readiness, and releases the lease.
  1. **Automatic Artifact Tainting**: `mu-cd-controller` sends `POST /api/v1/artifacts/<sha256>/taint` to `mu-registry` with the failure reason and failing stage/test identifier, preventing any future deployment of the defective artifact digest across the cluster.

______________________________________________________________________

### 3. Three-Phase Configuration Migration (`.muconde/` → `.mucd/`)

To adhere to the platform's Three-Phase Configuration Migration policy for dogfooded production services, continuous delivery configurations evolve across three strict phases:

1. **Phase 1: Dual-Stack Introduction & Backward-Compatible Fallback**:
   - `mu-cd-controller` and `mu-cd-agent` introduce native support for `.mucd/mu-cd.yml` (and `.mucd/mu-cd.yaml`) while retaining full fallback support for legacy `.muconde/MuConDe.yml` (and `.muconde/MuConDe.yaml`).
   - If a repository contains both directories, `.mucd/` takes precedence.
   - When resolving manifests from `.muconde/`, a structured `tracing::warn!` message is logged advising operators to migrate to `.mucd/mu-cd.yml`.
   - Side-by-side configurations allow immediate adoption without disrupting operational nodes.
1. **Phase 2: Production Deployment & Cluster Verification**:
   - The upgraded `mu-cd-controller` and `mu-cd-agent` daemons are deployed across production hosts (`node-a`, `node-b`, `worker-riscv64`).
   - Repositories migrate their delivery manifests from `.muconde/MuConDe.yml` to `.mucd/mu-cd.yml`.
   - Live continuous delivery cycles verify seamless deployment, watchdog health probes, and canary bake-window gates under the new `.mucd/` manifest scheme.
1. **Phase 3: Deprecation & Legacy Fallback Decommissioning**:
   - Once all services and repositories have successfully adopted `.mucd/mu-cd.yml` and verified multiple green deployments, legacy fallback code and references to `.muconde/` are removed from `mu-cd-controller` and `mu-cd-agent`.

## Consequences

- Zero in-flight CI jobs or model evaluation passes are interrupted during continuous delivery (`active_slots == 0` invariant).
- Rollback completes in milliseconds via atomic symlink swap + Btrfs read-only snapshot restoration without touching the host OS rootfs.
- Defective release artifacts are automatically quarantined in `mu-registry` via SHA-256 tainting.
- Configuration schemas migrate monotonically without breaking running continuous delivery workflows.
