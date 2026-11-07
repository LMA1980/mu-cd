______________________________________________________________________

## name: mu-cd description: "Orchestrate zero-downtime A/B continuous delivery, Btrfs subvolume snapshots, atomic rollback, and canary deployment verification across mu-cd-controller, mu-cd-agent, and mu-cd-ipc."

# `mu-cd` Zero-Downtime Continuous Delivery Workspace

## 1. Per-Service A/B Btrfs Subvolume Lifecycle

- Continuous delivery operates strictly on per-service Btrfs subvolumes (`/srv/mucd/<service>/@a`, `/srv/mucd/<service>/@b`) and immutable read-only snapshots (`/srv/mucd/<service>/snapshots/@snap-<prev_digest>`).
- Full-OS root filesystem switching is strictly forbidden; only allowlisted services (`mu-ci-worker`, `mu-ci-controller`, `mu-registry`, `mu-cd-agent`, `jero-runner`) are managed.
- Atomic slot activation is performed by creating a temporary symlink (`.current.tmp`) and executing POSIX `rename()` to `/srv/mucd/<service>/current`.

## 2. Idle-Worker Detection & Lease Coordination

- Before updating active symlinks or restarting services, `mu-cd-controller` acquires a deployment lease (`AcquireLeaseRequest`) from the target `mu-cd-agent`.
- Nodes grant leases only when `active_slots == 0` (no active CI stages or inference passes running).
- Nodes are cordoned from new job dispatches for the duration of `lease_ttl_seconds` (default `300s`).

## 3. Canary Bake-Window SLO & Automatic Tainting

- Every deployment specification (`.mucd/mu-cd.yml` or fallback `.muconde/MuConDe.yml`) enforces a canary warmup and bake window with strict SLO gates (`max_error_logs: 0`, `max_p99_latency_ms: 250.0`).
- If an SLO breach, health probe failure, or timeout occurs, `mu-cd-agent` executes an instant atomic rollback (`RollbackSlotRequest`) to the previous slot/snapshot, and `mu-cd-controller` issues an artifact taint request to `mu-registry`.

## 4. Three-Phase Configuration Migration

- **Phase 1 (Dual-Stack)**: Daemons check `.mucd/mu-cd.yml` first and fall back to legacy `.muconde/MuConDe.yml` with a deprecation warning.
- **Phase 2 (Cluster Verification)**: Production hosts deploy updated daemons and migrate repository configs to `.mucd/mu-cd.yml`.
- **Phase 3 (Decommissioning)**: Legacy `.muconde/` fallback code is removed once all services are migrated.
