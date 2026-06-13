# Service Inventory

## Legend
- **Host:** `pi5` = Raspberry Pi 5, `vps` = Cloud VPS, `garuda` = Garuda Arch Linux Desktop
- **Mount:** All paths under `/mnt/lvm` are on the external 6TB LVM volume
- **Net:** Network namespace / Docker network the container is attached to

---

## Primary Node — Raspberry Pi 5 (`pi5`)

| Service | Type | Port(s) | Storage Path | Net | Notes |
|---------|------|---------|-------------|-----|-------|
| Homepage | Dashboard (deprecated) | `[PLACEHOLDER]:3000` | `./homepage/config` | `infra_net` | Replaced by Dashy on VPS — kept for local fallback |
| Portainer CE | Management | `[PLACEHOLDER]:9000` | `./portainer/data` | `infra_net` | Agentless multi-cluster management |
| Nextcloud | File sync | `[PLACEHOLDER]:8080` | `/mnt/lvm/nextcloud/data`, `./nextcloud/db` | `media_net` | High-availability productivity suite |
| Immich | Photo backup | `[PLACEHOLDER]:2283` | `/mnt/lvm/immich/photos`, `./immich/db`, `./immich/config` | `media_net` | Auto mobile photo/video backup |
| Jellyfin | Media server | `[PLACEHOLDER]:8096` | `/mnt/lvm/media:/media:ro` | `media_net` | Read-only mount on media |
| Prowlarr | Indexer | `[PLACEHOLDER]:9696` | `./prowlarr/config` | `media_net` | Media indexer |
| Radarr | Movie management | `[PLACEHOLDER]:7878` | `./radarr/config` | `media_net` | Movie lifecycle |
| qBittorrent | Torrent client | `[PLACEHOLDER]:8080` (Web UI) | `./qbittorrent/config`, `/mnt/lvm/downloads` | `gluetun` | Bound to Gluetun namespace |
| Gluetun | VPN gateway | N/A | `./gluetun/config` | `gluetun` | Surfshark WireGuard/OpenVPN |
| Uptime Kuma | Monitoring | `[PLACEHOLDER]:3001` | `./uptime-kuma/data` | `infra_net` | HTTP/TCP/Ping monitors |
| Matrix Synapse | Chat server (E2EE) | `[PLACEHOLDER]:8008` | `./synapse/data`, `./synapse/config` | `media_net` | Secure, decentralized messaging with friends |
| Matrix Postgres (synapse) | Database | — | `./synapse/db` | `media_net` | PostgreSQL 15 Alpine for Synapse |
| Matrix RTC JWT | Auth token service | `[PLACEHOLDER]:8081` | `./synapse/rtc-jwt` | `media_net` | JWT authentication for LiveKit calls |
| LiveKit Server | WebRTC SFU | `[PLACEHOLDER]:7880` (TCP), `[PLACEHOLDER]:7881` (TCP), `[PLACEHOLDER]:50100-50200` (UDP) | `./synapse/livekit` | `media_net` | Voice/video call infrastructure for Element X |

---

## Cloud VPS (`vps`)

| Service | Type | Port(s) | Storage | Notes |
|---------|------|---------|---------|-------|
| WireGuard | VPN server | `[PLACEHOLDER]`/UDP | N/A | WireGuard server for LAN peer connectivity |
| Pi-hole | DNS sinkhole | `53/UDP`, `53/TCP` | `./pihole/config`, `./pihole/dnsmasq` | Network-wide ad-blocking, DNS for all LAN devices via WireGuard tunnel |
| Dashy | Dashboard (Homepage) | `[PLACEHOLDER VPS IP]:8080` | `./dashy/config` | Centralized services homepage, replaces Homepage.dev |
| Caddy | Reverse proxy | `443/TCP`, `80/TCP` | `./caddy/data`, `./caddy/config` | Terminates HTTPS, proxies to Pi services over WireGuard |
| Vault Warden | Password manager | `[PLACEHOLDER VPS IP]:8989` | `./vaultwarden/data` | Self-hosted Bitwarden-compatible password manager |

---

## Garuda Arch Linux Desktop (`garuda`)

| Service | Type | Port(s) | Storage | Notes |
|---------|------|---------|---------|-------|
| Ollama | LLM runtime | `[PLACEHOLDER]:11434` | `./ollama/models` | GPU-accelerated inference |
| Open WebUI | Chat frontend | `[PLACEHOLDER]:3000` | `./open-webui/data` | Connects to Ollama API |
