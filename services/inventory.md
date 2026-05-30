# Service Inventory

## Legend
- **Host:** `pi5` = Raspberry Pi 5, `vps` = Cloud VPS, `garuda` = Garuda Arch Linux Desktop
- **Mount:** All paths under `/mnt/lvm` are on the external 6TB LVM volume
- **Net:** Network namespace / Docker network the container is attached to

---

## Primary Node — Raspberry Pi 5 (`pi5`)

| Service | Type | Port(s) | Storage Path | Net | Notes |
|---------|------|---------|-------------|-----|-------|
| Homepage | Dashboard | `[PLACEHOLDER]:3000` | `./homepage/config` | `infra_net` | Unified app portal (gethomepage.dev) |
| Portainer CE | Management | `[PLACEHOLDER]:9000` | `./portainer/data` | `infra_net` | Agentless multi-cluster management |
| Nextcloud | File sync | `[PLACEHOLDER]:8080` | `/mnt/lvm/nextcloud/data`, `./nextcloud/db` | `media_net` | High-availability productivity suite |
| Immich | Photo backup | `[PLACEHOLDER]:2283` | `/mnt/lvm/immich/photos`, `./immich/db`, `./immich/config` | `media_net` | Auto mobile photo/video backup |
| Jellyfin | Media server | `[PLACEHOLDER]:8096` | `/mnt/lvm/media:/media:ro` | `media_net` | Read-only mount on media |
| Prowlarr | Indexer | `[PLACEHOLDER]:9696` | `./prowlarr/config` | `media_net` | Media indexer |
| Radarr | Movie management | `[PLACEHOLDER]:7878` | `./radarr/config` | `media_net` | Movie lifecycle |
| qBittorrent | Torrent client | `[PLACEHOLDER]:8080` (Web UI) | `./qbittorrent/config`, `/mnt/lvm/downloads` | `gluetun` | Bound to Gluetun namespace |
| Gluetun | VPN gateway | N/A | `./gluetun/config` | `gluetun` | Surfshark WireGuard/OpenVPN |
| Uptime Kuma | Monitoring | `[PLACEHOLDER]:3001` | `./uptime-kuma/data` | `infra_net` | HTTP/TCP/Ping monitors |

---

## Cloud VPS (`vps`)

| Service | Type | Port(s) | Storage | Notes |
|---------|------|---------|---------|-------|
| WireGuard | VPN server | `[PLACEHOLDER]`/UDP | N/A | WireGuard server for LAN peer connectivity |
| Pi-hole | DNS sinkhole | `53/UDP`, `53/TCP` | `./pihole/config`, `./pihole/dnsmasq` | Network-wide ad-blocking, DNS for all LAN devices via WireGuard tunnel |
| Caddy | Reverse proxy | `443/TCP`, `80/TCP` | `./caddy/data`, `./caddy/config` | Terminates HTTPS, proxies to Pi services over WireGuard |

---

## Garuda Arch Linux Desktop (`garuda`)

| Service | Type | Port(s) | Storage | Notes |
|---------|------|---------|---------|-------|
| Ollama | LLM runtime | `[PLACEHOLDER]:11434` | `./ollama/models` | GPU-accelerated inference |
| Open WebUI | Chat frontend | `[PLACEHOLDER]:3000` | `./open-webui/data` | Connects to Ollama API |
