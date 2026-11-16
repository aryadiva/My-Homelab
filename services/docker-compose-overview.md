# Docker Compose Overview — Architecture Reference (Synced 2026-11-16)

> This document describes the current compose stack architecture and provides representative configuration snippets for reference.  
> ⚠️ **Important:** Exact container versions, image tags, port mappings, and volume paths may vary and are subject to change per user decision. Verify against `docker ps`, `docker-compose -f ... config` before using any config here in production.

---

## Architecture

Services run on two Docker Engine hosts connected via WireGuard (`wg0`). Each host has its own bridge networks (e.g., Pi uses `infra_net`, `media_net`; VPS uses `vps_net`).

### Raspberry Pi 5 (Docker Host) — Three Zones

| Zone Name    | Network Type | Purpose                               | Example Services                            |
|--------------|--------------|---------------------------------------|---------------------------------------------|
| `infra_net`  | bridge       | Management interfaces                 | Portainer Server, (deprecated Homepage)     |
| `media_net`  | bridge       | Media, storage & user-facing apps     | Nextcloud, Immich, Jellyfin, Matrix, etc.   |
| `gluetun`    | host+        | VPN-isolated torrent client           | Gluetun (NET_ADMIN), qBittorrent            |

### Cloud VPS (Docker Host) — One Zone + Native WireGuard

| Zone Name    | Network Type | Purpose                                    | Example Services                               |
|--------------|--------------|--------------------------------------------|------------------------------------------------|
| `vps_net`    | bridge       | Infrastructure services                    | Pi-hole, Dashy, Vaultwarden, Uptime Kuma      |
| wireguard0   | host-native  | Overlay server to all peers                | WireGuard (no container — OS-level service)    |

### Orange Pi 5 Pro (`bifrost_net`)

- **Bifrost** v2.3.2 and **SearXNG** live on this single network
- Used for MCP endpoint access and AI routing between local inference and hosted providers
---

## Representative Compose Snippets — For Reference Only

The following are historical examples of service configs — DO NOT copy directly into production without verification against live containers and security requirements (secrets, mounts).

### Nextcloud (Example — verify images and paths before use)

```yaml
# Example structure only — exact versions unknown to KB
nextcloud:
  image: nextcloud:apache   # Historically v32-apache
  container_name: nextcloud
  ports:
    - "8080:80"             # Bind on Pi's WG IP, verify current exposure
  volumes:
    - /mnt/lvm/nextcloud-data:/var/www/html/data         # Corrected path per KB
    - ./nextcloud/config:/var/www/html/config
    - ./nextcloud/apps:/var/www/html/custom_apps
  depends_on:
    - nextcloud-db
    - nextcloud-redis
```

### Immich (Multi-container Example)

The original stack includes microservices and ML components. Exact image versions vary; verify before redeploy:

```yaml
immich-server:
  image: ghcr.io/immich-app/immich-server:release-tag
  volumes:
    - /mnt/lvm/immich-app:/usr/src/app/upload   # Verify actual path used
    - ./immich/config:/usr/src/app/config
environment & dependencies:
  DB_HOSTNAME: immich-db
  REDIS_HOSTNAME: immich-redis
```

### Matrix Synapse (Multi-container Stack)

```yaml
synapse:
  container_name: synapse
  image: matrixdotorg/synapse:latest   # Verify version
  depends_on: &depends
    - synapse-db
  volumes:
    - ./synapse/data:/data
environment:
  SYNAPSE_SERVER_NAME: matrix.aryadivap.com
```

See `procedures/deployment.md` for standard update/rollback workflows.

---

## Network Definitions — Per Host

### Raspberry Pi (docker-compose.yml)

```yaml
networks:
  infra_net:
    driver: bridge
    ipam:
      config:
        - subnet: "172.20.0.0/16"
  media_net:
    driver: bridge
    ipam:
      config:
        - subnet: "172.21.0.0/16"

# gluetun shares namespace, no dedicated network entry needed for qBittorrent
```

### VPS (docker-compose.yml)

```yaml
networks:
  vps_net:
    driver: bridge
    ipam:
      config:
        - subnet: "172.22.0.0/16"

# WireGuard runs host-native; no compose network needed for WG service
```

---
