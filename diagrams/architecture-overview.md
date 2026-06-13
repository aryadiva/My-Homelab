# Architecture Diagram Walkthrough

This document provides a narrative walkthrough of the network diagram in [`Updated Homelab Architecture v3.drawio.png`](./Updated%20Homelab%20Architecture%20v3.drawio.png).

---

## What the Diagram Shows

The diagram visualizes the current hybrid-cloud topology:

### North-South Traffic (Internet → Service via VPS)

```
User → Cloudflare DNS (A record → VPS public IP)
    │
    ▼
Cloud VPS (public edge)
    │
    ├── Caddy reverse proxy (443/80)
    │     └── Terminates HTTPS, proxies via WireGuard
    │
    ├── Dashy Dashboard (8080, VPS IP)
    │     └── Central web portal for self-hosted services (replaces Homepage)
    │
    ├── Vault Warden (8989, VPS IP)
    │     └── Self-hosted Bitwarden-compatible password manager
    │
    ├── Pi-hole DNS (53)
    │     └── Resolves/filters DNS for all LAN devices
    │
    └── WireGuard server (wg0, 10.9.0.1)
          │
          ├── Raspberry Pi 5 (client, 10.9.0.4)
          │     ├── Docker Engine
          │     │     ├── Infrastructure (Portainer, Homepage, Uptime Kuma)
          │     │     ├── Media/Storage (Nextcloud, Immich, Jellyfin, Radarr, Prowlarr, Matrix Synapse, LiveKit)
          │     │     └── VPN-Isolated (Gluetun → qBittorrent)
          │     │
          │     └── LVM Storage (/mnt/lvm, 2×3TB HDDs)
          │
          ├── Garuda Arch Desktop (client, 10.9.0.2)
          │     └── Ollama + Open WebUI (GPU)
          │
          └── LAN devices (clients, laptops, phones)
```

### East-West Traffic (LAN Internal)

```
LAN device ──WireGuard── VPS (server) ──WireGuard── Pi/Garuda
                                      │
                                  Pi-hole DNS
```

---

## Key Elements in the Diagram

| Diagram Element | Real-World Counterpart | Notes |
|----------------|----------------------|-------|
| Public VPS node | Cloud VPS instance | WireGuard server, Caddy reverse proxy, Pi-hole DNS |
| Raspberry Pi 5 | On-premises compute | ARM64, 8GB, 6TB storage, WireGuard client |
| LVM storage pool | `/mnt/lvm` | 2×3TB HDDs combined via LVM |
| Docker containers (Pi) | All services | Split across 3 network zones |
| Docker containers (VPS) | Pi-hole + Caddy | DNS and reverse proxy |
| WireGuard icon | `wg0` server/client | `10.9.0.0/24` subnet, VPS is server |
| Tailscale icon | Fallback mesh | Out-of-band admin access |
| ER605 router | TP-Link Omada ER605 | Dual-ISP load balancing, NO port forwarding |
| Garuda node | Arch Linux desktop | GPU compute (Ollama), WireGuard client |
| Surfshark | Commercial VPN provider | Gluetun → qBittorrent only |
| Caddy | Reverse proxy on VPS | Proxies subdomains → Pi over WireGuard |

---

## Data Flow Examples

### 1. User accesses Nextcloud from the Internet

```
User types nextcloud.example.com
  │
  │ Cloudflare DNS: A record → VPS public IP
  ▼
Cloud VPS (port 443)
  │
  │ Caddy matches nextcloud.example.com
  │ Caddyfile: reverse_proxy http://10.9.0.4:8080
  │
  │ WireGuard tunnel (VPS → Pi)
  ▼
Raspberry Pi 5 (10.9.0.4)
  │
  │ Docker (media_net)
  ▼
Nextcloud container (port 8080)
  │
  ├── PostgreSQL (nextcloud-db)
  ├── Redis (nextcloud-redis)
  └── /mnt/lvm/nextcloud/data (file storage)
```

### 2. LAN device resolves DNS

```
Laptop → DNS server set to 10.9.0.1 (VPS WireGuard IP)
  │
  │ WireGuard tunnel to VPS
  ▼
VPS WireGuard server → Pi-hole container (10.9.0.x:53)
  │
  ├── If local domain: returns rewritten IP
  └── If internet domain: forwards to upstream DNS
```

### 3. qBittorrent downloads a torrent

```
qBittorrent container
    │
    │ (shared network namespace)
    ▼
Gluetun container
    │
    │ Surfshark WireGuard tunnel
    ▼
Internet (via VPN exit node)
    │
    ▼
Download written to /mnt/lvm/downloads/
```

### 4. Local file access via LAN (mobile device connected to WireGuard)

```
Phone (connected to WireGuard, DNS = 10.9.0.1)
  │
  │ Pi-hole resolves nextcloud.example.com → VPS public IP
  │ or using local domain → 10.9.0.x
  ▼
Cloud VPS → Caddy → WireGuard tunnel
  │
  ▼
Raspberry Pi 5 → Nextcloud
  │
  ▼
/mnt/lvm/nextcloud/data
```

---

## Suggested Improvements for the Diagram

These are observations for enhancing the existing `drawio.png` — no changes have been made to the file.

| # | Suggestion | Rationale |
|---|-----------|-----------|
| 1 | **Label overlay network subnets** (e.g., "WireGuard 10.9.0.0/24") | Makes the tunnel boundaries explicit |
| 2 | **Add a storage subsystem box** representing the 2×3TB HDDs + LVM | Currently storage is implicit; showing the LVM abstraction would clarify the disk layout |
| 3 | **Show the Gluetun isolation boundary** as a dashed box around qBittorrent | Highlights the VPN namespace model vs regular bridge containers |
| 4 | **Indicate port numbers** next to each container | Quick reference for which service runs on which port |
| 5 | **Add Tailscale as a separate cloud icon** with a dashed line labeled "fallback" | Clarifies that Tailscale is a secondary path, not primary |
| 6 | **Show Garuda Desktop** as a LAN-connected node with a GPU icon and "Ollama" label | Currently the diagram may not account for the AI compute offloading node |
| 7 | **Add a legend** for line styles (solid = wire, dashed = overlay, dotted = admin) | Reduces ambiguity about connection types |
| 8 | **Add VPS services box** showing Caddy + Pi-hole | Currently the VPS role is underrepresented in the diagram |
| 9 | **Show WireGuard as host-native on VPS** (not inside Docker) | Clarifies that WG runs at the system level, not containerized |
| 10 | **Label ER605 as "Load Balancing Only — No Port Forwarding"** | Important architectural detail for security understanding |

---

> **Note:** The actual `.drawio.png` file is maintained separately. If you want me to generate a new `.drawio` editable source from these notes, I can create a blank `network.drawio` file for you to edit in diagrams.net.
