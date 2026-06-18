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
    ├── Uptime Kuma (3001, VPS IP)
    │     └── HTTP/TCP/Ping monitoring for all services
    │
    ├── Pi-hole DNS (53)
    │     └── Resolves/filters DNS for all LAN devices
    │
    ├── Portainer (9000, VPS IP)
    │     └── Container management for VPS Docker
    │
    └── WireGuard server (wg0, 10.9.0.1)
          │
          ├── Raspberry Pi 5 (client, 10.9.0.4)
          │     ├── Docker Engine
          │     │     ├── Infrastructure (Portainer, Homepage)
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
| Docker containers (VPS) | Pi-hole, Caddy, Dashy, Vault Warden, Portainer, Uptime Kuma | DNS, reverse proxy, and management services |
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
