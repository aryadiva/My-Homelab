# Service Inventory — Synced 2026-11-16

> Historical services retained; KB marks them as needing live verification.
> Legend: **Confirmed** = current KB scope. **Historical** = previous docs, not re-verified.

---

## Raspberry Pi 5 (`pi5`) — Primary Compute Node

| Service | Type | Ports | Storage / Paths | Network | Status |
|---------|------|-------|-----------------|---------|--------|
| Portainer Server | Docker control plane | `10.9.0.4:9000` | `./portainer/data` | `infra_net` | **Confirmed** (main mgmt plane) |
| Nextcloud | File sync cloud | `10.9.0.4:8080` | `/mnt/lvm/nextcloud-data` | `media_net` | Confirmed (`nextcloud:32-apache`, MariaDB+Redis) |
| MariaDB (`mariadb:lts`) | Nextcloud DB | — | `./nextcloud/db` | `media_net` | Confirmed |
| Redis (`redis:alpine`) | Nextcloud cache | — | — | `media_net` | Confirmed |
| Immich | Photo backup | `10.9.0.4:2283` | `/mnt/lvm/immich-app` (verify — old docs say `/mnt/lvm/immich/photos`) | `media_net` | Confirmed |
| Jellyfin | Media server | `10.9.0.4:8096` | `/mnt/lvm/media:/media:ro` | `media_net` | Confirmed |
| Prowlarr | Indexer | `10.9.0.4:9696` | `./prowlarr/config` | `media_net` | Confirmed |
| Radarr | Movie lifecycle | `10.9.0.4:7878` | `./radarr/config`, `/mnt/lvm/downloads`, `/mnt/lvm/media/movies` | `media_net` | Confirmed |
| qBittorrent | Torrent client | `10.9.0.4:8080` (WebUI) | `./qbittorrent/config`, `/mnt/lvm/downloads` | via Gluetun | Confirmed — **port conflict with Nextcloud `:8080` noted; verify** |
| Gluetun | VPN gateway (Surfshark) | N/A | `./gluetun/config` | host / NET_ADMIN | Confirmed |
| Matrix Synapse | Chat server (E2EE) | `10.9.0.4:8008` | `./synapse/data`, `./synapse/config` | `media_net` | Confirmed |
| Matrix Auth + Matrix DB + RTC JWT | Matrix stack | various | `./synapse/*` | `media_net` | Confirmed (multi-container) |
| LiveKit | WebRTC SFU | `10.9.0.4:7880/7881 TCP`, `50100-50200 UDP` | `./synapse/livekit` | `media_net` | Confirmed |
| Homepage | Dashboard (deprecated) | — | `./homepage/config` | `infra_net` | Historical — replaced by Dashy on VPS |

---

## Orange Pi 5 Pro (`orangepi`) — AI Gateway Node

| Service | Type | Ports / Endpoint | Storage | Network | Status |
|---------|------|------------------|---------|---------|--------|
| Bifrost | AI gateway (OpenAI-compatible) | `https://bifrost.aryadivap.com`, Docker version `v2.3.2` | — | `bifrost_net` | Confirmed (v2.3.2) |
| SearXNG | Meta search engine | JSON output tested | — | `bifrost_net` | Confirmed |
| Authentik | Identity / SSO | — | — | — | **Uncertain** — installed/explored; may be disabled |
| Forgejo | Git hosting | — | — | — | Historical (2026-11-06 inventory) |
| Time-MCP | Time service | — | — | — | Historical (2026-11-06 inventory) |

MCP endpoint previously tested: `https://bifrost.aryadivap.com/mcp/core-mcp`.

---

## Cloud VPS (`vps`) — Public Edge

| Service | Type | Ports | Storage | Status |
|---------|------|-------|---------|--------|
| WireGuard (host-native) | VPN server | `51820/udp`, `wg0` `10.9.0.1/24` | N/A | Confirmed |
| UFW | Host firewall | — | — | Confirmed (default deny incoming) |
| Caddy | Reverse proxy / TLS | `0.0.0.0:80,443 TCP+UDP` | `./caddy/data`, `./caddy/config` | Confirmed |
| CrowdSec | Intrusion detection | — | — | Confirmed (verify bouncer integration) |
| fail2ban | Intrusion remediation | — | — | Confirmed |
| Pi-hole | DNS sinkhole | `10.9.0.1:53 TCP+UDP` | `./pihole/config`, `./pihole/dnsmasq` | Confirmed — VPS only, accessed over WireGuard |
| Uptime Kuma | Monitoring | `10.9.0.1:3001` | `./uptime-kuma-vps/data` | Confirmed |
| Vaultwarden | Password manager (Bitwarden-compatible) | `10.9.0.1:8989` | `./vaultwarden/data` | Confirmed |
| Dashy | Dashboard | `10.9.0.1:8080` | `./dashy/config` | Confirmed (replaces deprecated Homepage) |
| Portainer CE (VPS instance) | Docker management | `10.9.0.1:9000` | `./portainer-vps/data` | Historical — not in current KB scope; verify |
