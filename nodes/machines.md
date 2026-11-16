# Machine Inventory

> Synced with knowledge base snapshot **2026-11-16**. Entries marked "historical" were accurate at some
> earlier point and have not been re-verified. Do not treat inferred roles as final — see
> [Open Questions](#open-questions).

---

## 1. Raspberry Pi 5 Model B — 8 GB (`pi5`)

| Item | Value | Confidence |
|------|-------|------------|
| Architecture | ARM64 (Model B) | Confirmed |
| RAM | 8 GB | Confirmed |
| OS | Ubuntu Server | Confirmed |
| Container engine | Docker Engine | Confirmed |
| Role | Personal cloud, photos, media streaming, Matrix stack, media acquisition; **Portainer Server control plane** | Confirmed |
| Storage | External HDD managed with LVM (VG `vg_nextcloud`, LV `lv_data`) | Confirmed |
| Memory snapshot | ~7.7 GiB total, ~4.6 GiB available, swap 0, ~3.2 GiB Docker usage | Historical |

Interesting fact: exact disk capacity, filesystem, mount options and free space are not recorded — verify with `lsblk` / `df -h` before storage changes. **No RAID is desired.**

## 2. Orange Pi 5 Pro — 4 GB (`orangepi`)

| Item | Value | Confidence |
|------|-------|------------|
| Architecture | ARM64 | Confirmed |
| RAM | 4 GB | Confirmed |
| Role | Bifrost AI gateway + infrastructure/utility services. Final long-term role undecided | Confirmed |
| Docker network | `bifrost_net` (Bifrost + SearXNG) | Confirmed |

## 3. H81H3-I x86 Mini PC (`h81`)

| Item | Value | Confidence |
|------|-------|------------|
| Motherboard | ECS H81H3-I (LGA1150, DDR3) | Confirmed |
| CPU | Intel Core i5-4570T (2C/4T) — **not** i5-4590 / i7 | Confirmed (corrected) |
| RAM | 12 GB DDR3 | Confirmed (most recent statement) |
| Storage | 128 GB SATA SSD + external 2 TB HDD (SATA, PSU-powered) | Confirmed |
| OS | Ubuntu Server (Ventoy install once hit "need to load kernel first"; machine later booted normally — not a hardware fault) | Confirmed |
| Role | **Unsettled.** Candidates: production x86 host, test host, k3s practice node | Open |

Suitability: conventional x86 Docker/Compose, light databases, monitoring, reverse-proxy workloads, k3s
learning. Not suitable for heavy local LLM inference. The 2 TB HDD is bulk storage only — no redundancy,
no RAID, not a backup.

## 4. Cloud VPS (`vps`)

| Item | Value | Confidence |
|------|-------|------------|
| Role | Public edge for CGNAT home network: WireGuard server, reverse proxy/TLS, firewall, security enforcement, light infra services | Confirmed |
| WireGuard | `wg0`, `10.9.0.1/24`, listen `51820/udp` | Confirmed |
| Firewall | UFW — see `network/firewall-rules.md` | Confirmed |

Hosts: WireGuard, UFW, Caddy, CrowdSec, fail2ban, Pi-hole, Uptime Kuma, Vaultwarden, Dashy.

## 5. AMD Desktop Workstation — Local AI Host (`garuda`)

| Item | Value | Confidence |
|------|-------|------------|
| CPU | Intel Core i7-10700K | Confirmed |
| RAM | 32 GB DDR4 | Confirmed |
| Motherboard | MSI Z490M Gaming Edge WiFi | Confirmed |
| GPU | Sapphire Pure AMD Radeon RX 9060 XT 16 GB (upgraded from RTX 3060 Ti 8 GB), `gfx1200` | Confirmed |
| PSU | MSI A650GF — **not** A650GL | Confirmed (corrected) |
| Cooling | Cooler Master ML240L V2 (~5 years old) | Confirmed |
| OS | Garuda Linux (Arch-based), KDE Plasma / Wayland, 1440p @ 200 Hz | Confirmed |
| Storage | Samsung 970 EVO Plus 500 GB (OS), MSI M460 1 TB (games) | Confirmed |
| AI role | Local model serving via **custom llama.cpp ROCm container** (built from source, ROCm 7.2.x). Ollama is **not** the active server — port `11434` was only investigated historically | Confirmed |
| Benchmark | ~45–46 tok/s baseline, ~2 min/test generation, ~41% VRAM observed, PLAN: GPU power DPM high | Historical |

Not a general-purpose always-on server — reachability must be confirmed before relying on it.

## 6. HP Spectre x360 — **NOT a homelab node** (`spectre`)

| Item | Value |
|------|-------|
| Model | HP Spectre x360 Convertible 13-ap0xxx |
| OS | Zorin OS 18.1 on 256 GB NVMe |
| CPU / RAM | i7-8565U / 16 GB DDR4, touchscreen |
| Purpose | Personal laptop — display, media, presentation, tablet mode. **No server / test / k3s role, ever** |
| WireGuard use | Private network access + VPS Pi-hole (ad blocking) only |

---

## Open Questions

- Final role for H81 — user is still evaluating production / test / k3s placement.
- Final role for Orange Pi — AI gateway only vs general production node.
- Exact WireGuard peer inventory, AllowedIPs, DNS settings — verify against live `wg show`.
- Whether all historical services remain deployed.

See also: `services/inventory.md`, `network/ip-assignments.md`, `services/migration-plan.md`.
