# Service Inventory

## Legend
- **Host:** `pi5` = Raspberry Pi 5, `vps` = Cloud VPS, `garuda` = Garuda Arch Linux Desktop
- **Mount:** All paths under `/mnt/lvm` are on the external 6TB LVM volume
- **Net:** Network namespace / Docker network the container is attached to

---

## Primary Node — Raspberry Pi 5 (`pi5`)

| Service | Type | Port(s) | Storage Path | Net | Notes |
|---------|------|---------|-------------|-----|-------|
| Homepage | Dashboard (deprecated) | `10.9.0.4:3000` | `./homepage/config` | `infra_net` | Replaced by Dashy on VPS — kept for local fallback |
| Portainer CE (Pi) | Management | `10.9.0.4:9000` | `./portainer/data` | `infra_net` | Agentless multi-cluster management (Pi instance) |
| Nextcloud | File sync | `10.9.0.4:80` (Caddy) / `10.9.0.4:8080` (direct) | `/mnt/lvm/nextcloud/data`, `./nextcloud/db` | `media_net` | High-availability productivity suite |
| Immich | Photo backup | `10.9.0.4:2283` | `/mnt/lvm/immich/photos`, `./immich/db`, `./immich/config` | `media_net` | Auto mobile photo/video backup |
| Jellyfin | Media server | `10.9.0.4:8096` | `/mnt/lvm/media:/media:ro` | `media_net` | Read-only mount on media |
| Prowlarr | Indexer | `10.9.0.4:9696` | `./prowlarr/config` | `media_net` | Media indexer |
| Radarr | Movie management | `10.9.0.4:7878` | `./radarr/config` | `media_net` | Movie lifecycle |
| qBittorrent | Torrent client | `10.9.0.4:8080` (Web UI) | `./qbittorrent/config`, `/mnt/lvm/downloads` | `gluetun` | Bound to Gluetun namespace |
| Gluetun | VPN gateway | N/A | `./gluetun/config` | `gluetun` | Surfshark WireGuard/OpenVPN |
| Matrix Synapse | Chat server (E2EE) | `10.9.0.4:8008` | `./synapse/data`, `./synapse/config` | `media_net` | Secure, decentralized messaging with friends |
| Matrix Postgres (synapse) | Database | — | `./synapse/db` | `media_net` | PostgreSQL 15 Alpine for Synapse |
| Matrix RTC JWT | Auth token service | `10.9.0.4:8081` | `./synapse/rtc-jwt` | `media_net` | JWT authentication for LiveKit calls |
| LiveKit Server | WebRTC SFU | `10.9.0.4:7880` (TCP), `10.9.0.4:7881` (TCP), `10.9.0.4:50100-50200` (UDP) | `./synapse/livekit` | `media_net` | Voice/video call infrastructure for Element X |

---

## Cloud VPS (`vps`)

| Service | Type | Port(s) | Storage | Notes |
|---------|------|---------|---------|-------|
| WireGuard | VPN server | `51820`/UDP | N/A | WireGuard server for LAN peer connectivity (host-native, not Docker) |
| Pi-hole | DNS sinkhole | `10.9.0.1:53/UDP`, `10.9.0.1:53/TCP` | `./pihole/config`, `./pihole/dnsmasq` | Network-wide ad-blocking, DNS for all LAN devices via WireGuard tunnel |
| Dashy | Dashboard | `10.9.0.1:8080` | `./dashy/config` | Centralized services homepage, replaces Homepage.dev |
| Caddy | Reverse proxy | `0.0.0.0:443/TCP`, `0.0.0.0:80/TCP` | `./caddy/data`, `./caddy/config` | Terminates HTTPS, proxies to Pi services over WireGuard |
| Vault Warden | Password manager | `10.9.0.1:8989` | `./vaultwarden/data` | Self-hosted Bitwarden-compatible password manager |
| Portainer CE (VPS) | Management | `10.9.0.1:9000` | `./portainer-vps/data` | Agentless multi-cluster management (VPS instance) |
| Uptime Kuma | Monitoring | `10.9.0.1:3001` | `./uptime-kuma-vps/data` | HTTP/TCP/Ping monitors for VPS services |

---

## Garuda Arch Linux Desktop (`garuda`)

| Service | Type | Port(s) | Storage | Notes |
|---------|------|---------|---------|-------|
| Ollama | LLM runtime | `10.9.0.2:11434` | `./ollama/models` | GPU-accelerated inference |
| Open WebUI | Chat frontend | `10.9.0.2:3000` | `./open-webui/data` | Connects to Ollama API |
