# My-Homelab Documentation — Synced 2026-11-16

> Current reference state from knowledge base `standards/homelab-ecosystem-2026-11-16-KNOWLEDGE-BASE.md`.
> Historical documentation (files below) was accurate previously but has not been re-verified.
> Treat historical values as "confirmed at some earlier point" until tested against live output.

---

## Table of Contents

0. [Quick Machine Summary](#quick-machine-summary) — 6 hosts, WireGuard overlay, CGNAT edge, no RAID
1. [Documentation Structure](#documentation-structure)
2. [Machine Inventory](./machines.md)
3. [Overview & Goals](./README.md#system-goals)
4. [Network Topology](./network/topology.md)
5. [IP Assignments](./network/ip-assignments.md) — WireGuard subnet `10.9.0.0/24`, VPS at `10.9.0.1`
6. [Firewall Rules](./network/firewall-rules.md) — UFW policy + container isolation
7. [Service Inventory](./services/inventory.md) — by host, with notes / historical tags
8. [Docker Compose Reference](./services/docker-compose-overview.md) — architecture and snippets (historical)
9. [Container Network Map](./services/container-network-map.md) — isolation model
10. [Migration Exploration: Docker → k3s](./services/migration-plan.md) — **proposed**, not executed
11. [Backup & Restore](./procedures/backup-restore.md) — blueprint, schedules TBD
12. [Deployment Guide](./procedures/deployment.md) — workflows / rollback references
13. [VPN Configuration](./procedures/vpn-config.md) — WireGuard server on VPS, Tailscale fallback (unverified), Gluetun

---

## Quick Machine Summary

| Machine | Role | Key Notes |
|---------|------|-----------|
| Raspberry Pi 5 (8 GB) | Personal cloud + Portainer Server control plane | LVM external HDD `/mnt/lvm`, MariaDB+Redis Nextcloud, Immich, Matrix stack, media acquisition. |
| Orange Pi 5 Pro (4 GB) | Bifrost AI gateway + utilities | Docker network `bifrost_net`. Final role unsettled. |
| H81H3-I mini PC (i5-4570T, 12 GB) | **Unsettled** — prod/test/k3s learning | Bulk 2 TB HDD stored on same SATA controller; no RAID. |
| Cloud VPS | Public edge (WireGuard server, Caddy, firewall) | CGNAT constraints apply. Hosts Pi-hole, Vaultwarden, Dashy, Uptime Kuma, CrowdSec, fail2ban. |
| AMD Desktop (Garuda, RX 9060 XT) | Local AI server — llama.cpp ROCm | NOT always-on server. Benchmarked ~45–46 tok/s baseline; PLAN harness comparison. |
| HP Spectre x360 | **Not a homelab node** | Personal laptop for access to VPS Pi-hole only. |

CGNAT constraint: home network behind provider CGNAT — outbound-only; all external ingress routes through VPS.

WireGuard subnet: `10.9.0.0/24`, VPS at `10.9.0.1/24` (verified via historical docs; peers need confirmation). Tailscale is unverified as deployed.

Pi-hole runs only on VPS, accessed over WireGuard (does NOT run on Pi).

---

## System Goals

- **Privacy-focused**, self-hosted where possible
- Reduce dependence on Google/Proton/Microsoft ecosystems
- Open standards and portability; low vendor lock-in
- Incremental changes with explicit rollback paths
- Document known decisions + mark uncertain areas

AI architecture separates model serving (llama.cpp ROCm on Garuda) from gateway routing (Bifrost `x-bf-vk` header).

---

## Open Questions & Known Uncertainties

The following are explicitly **not confirmed** and should be verified before making changes that affect them:

- H81, Orange Pi and Raspberry Pi final roles
- Peer IP inventory / AllowedIPs / DNS on WireGuard peers
- Whether Authentik is truly disabled or just uncertain in historical records
- Current Nextcloud version (32.0.11 reported pre-v33 exploration; outcome unknown)
- Correct database engine for Immich (current docs say PostgreSQL, KB marks uncertain)
- Exact Docker Compose configurations / service lists remain deployed

See `services/migration-plan.md` and `machines.md` sections "Open Questions".
