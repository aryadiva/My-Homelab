# Migration Exploration: Docker Compose → k3s — Synced 2026-11-16

> **Status:** This is a PROPOSAL, not an executed migration. The current stack remains Docker Compose.
k3s (Kubernetes light) is being investigated for DevOps learning and future-proofing.

---

## Current State (Verified)

- Primary deployment model: separate Docker Engine per host, managed via Portainer Server
- No Kubernetes cluster actively deployed or serving live workloads
- k3s exploration is in planning; no completed migration to date

See [`machines.md`](../machines.md) for hardware constraints on multi-node scheduling.

---

## Motivations and Constraints

| Factor | Detail |
|--------|--------|
| Learning goal | Practical DevOps exercise with real workloads |
| Multi-arch challenge | Orange Pi + Pi are `arm64`; H81 is `x86_64` — image availability must be checked per service |
| Resource limits | 8 GB (Pi), 4 GB (Orange Pi), 12 GB (H81) leave little headroom for control plane + workloads |
| Stateful services | Nextcloud, Immich, Matrix, media stacks hold critical data — migration needs explicit storage, backup, and rollback plans |
| Storage paths | Current LVM volumes (`/mnt/lvm/*`) must map cleanly into PV/PVC layers |

---

## Proposed Phases (Not Executed)

### Phase 0 — Pre-flight

- [ ] Verify current container inventory and health on each host
- [ ] Confirm backups exist and restore was tested (see `procedures/backup-restore.md`)
- [ ] Decide H81 final role (candidate k3s practice node vs production)
- [ ] Decide Orange Pi final role
- [ ] Verify multi-arch images for all services in scope

### Phase 1 — Control Plane Pilot (Proposed)

- [ ] Install k3s on H81 single node (x86_64)
- [ ] Deploy one stateless service as proof of concept
- [ ] Validate ingress (Traefik bundled or Caddy replacement) and Let's Encrypt
- [ ] Document resource usage and failure modes

### Phase 2 — Stateful Pilot (Proposed)

- [ ] Choose low-risk stateful service (possibly not Nextcloud initially)
- [ ] Provision PV/PVC on H81 storage or LVM
- [ ] Validate backup/restore inside k3s
- [ ] Compare operational complexity vs current Docker Compose

### Phase 3 — Decision

- [ ] Decide: stay Docker Compose, hybrid (k3s on H81 + Compose elsewhere), or full migration
- [ ] If migrating: prioritize Nextcloud + Immich data services with full pre-migration backups

---

## Rollback Strategy

- Keep existing Compose stacks intact during pilot phases
- Snapshot / backup LVM data before any data migration
- DNS / reverse proxy cutover is reversible by pointing Caddy routes back to Compose-hosted services
- Portainer Server on Pi remains the management plane until k3s proves out

---

## Known Risks

- Resource exhaustion on 4 GB / 8 GB ARM nodes if k3s agents are added there
- Multi-arch image gaps delay migrations
- Stateful migration errors can lose data — verify backups first
- Mixed cluster adds routing complexity (ARM and x86 nodes)

---
