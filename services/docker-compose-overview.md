# Docker Compose Overview
### ```This section is still not finished/finalized...``` 
## Architecture

Containers are organized into logical groups. Each group shares a Docker bridge network. Services are split across two hosts:

### Raspberry Pi 5 (Docker Host)

| Group | Network | Includes |
|-------|---------|----------|
| Infrastructure | `infra_net` | Homepage, Portainer |
| Media/Storage | `media_net` | Nextcloud, Immich, Matrix Synapse, LiveKit, Jellyfin, Prowlarr, Radarr |
| VPN-Isolated | `gluetun` | qBittorrent (shares Gluetun network namespace) |

### Cloud VPS (Docker Host)

| Group | Network | Includes |
|-------|---------|----------|
| VPS Services | `vps_net` | Pi-hole, Caddy, Dashy, Vault Warden, Portainer, Uptime Kuma |
| WireGuard | Host-level | WireGuard runs natively on the VPS (not in Docker) |

---

## Compose File Structure

Services are split across multiple Docker Compose files:
- **`infra.yml`** — Infrastructure services (Homepage, Portainer on Pi)
- **`media.yml`** — Media/storage services (Nextcloud, Immich, Matrix, LiveKit, Jellyfin, Prowlarr, Radarr on Pi)
- **`vpn.yml`** — VPN-isolated services (Gluetun + qBittorrent on Pi)
- **`vps.yml`** — VPS-hosted services (Pi-hole, Caddy, Dashy, Vault Warden, Portainer, Uptime Kuma on VPS)

Below are representative Compose snippets for each service. Adjust paths, versions, and environment variables to match your actual setup.

---

### Raspberry Pi — Infrastructure Network (`infra_net`)

#### Homepage (gethomepage.dev)

```yaml
homepage:
  image: ghcr.io/gethomepage/homepage:latest
  container_name: homepage
  ports:
    - "10.9.0.4:3000:3000"
  volumes:
    - ./homepage/config:/app/config
    - /var/run/docker.sock:/var/run/docker.sock:ro
  restart: unless-stopped
  networks:
    - infra_net
```

#### Portainer CE

```yaml
portainer:
  image: portainer/portainer-ce:latest
  container_name: portainer
  ports:
    - "10.9.0.4:9000:9000"
  volumes:
    - /var/run/docker.sock:/var/run/docker.sock:ro
    - ./portainer/data:/data
  restart: unless-stopped
  networks:
    - infra_net
```

---

### Raspberry Pi — Media / Storage Network (`media_net`)

#### Nextcloud (with PostgreSQL + Redis)

```yaml
nextcloud:
  image: nextcloud:stable
  container_name: nextcloud
  ports:
    - "10.9.0.4:8080:80"
  volumes:
    - /mnt/lvm/nextcloud/data:/var/www/html/data
    - ./nextcloud/config:/var/www/html/config
    - ./nextcloud/apps:/var/www/html/custom_apps
  environment:
    POSTGRES_HOST: nextcloud-db
    POSTGRES_DB: nextcloud
    POSTGRES_USER: nextcloud
    POSTGRES_PASSWORD: "[PLACEHOLDER]"
    REDIS_HOST: nextcloud-redis
  depends_on:
    - nextcloud-db
    - nextcloud-redis
  restart: unless-stopped
  networks:
    - media_net

nextcloud-db:
  image: postgres:15-alpine
  container_name: nextcloud-db
  volumes:
    - ./nextcloud/db:/var/lib/postgresql/data
  environment:
    POSTGRES_DB: nextcloud
    POSTGRES_USER: nextcloud
    POSTGRES_PASSWORD: "[PLACEHOLDER]"
  restart: unless-stopped
  networks:
    - media_net

nextcloud-redis:
  image: redis:7-alpine
  container_name: nextcloud-redis
  restart: unless-stopped
  networks:
    - media_net
```

#### Immich (microservices stack)

```yaml
immich-server:
  image: ghcr.io/immich-app/immich-server:release
  container_name: immich-server
  ports:
    - "10.9.0.4:2283:2283"
  volumes:
    - /mnt/lvm/immich/photos:/usr/src/app/upload
    - ./immich/config:/usr/src/app/config
  environment:
    DB_HOSTNAME: immich-db
    DB_USERNAME: immich
    DB_PASSWORD: "[PLACEHOLDER]"
    DB_DATABASE: immich
    REDIS_HOSTNAME: immich-redis
  depends_on:
    - immich-db
    - immich-redis
  restart: unless-stopped
  networks:
    - media_net

immich-microservices:
  image: ghcr.io/immich-app/immich-server:release
  container_name: immich-microservices
  entrypoint: ["node", "/usr/src/app/dist/microservices/main"]
  volumes:
    - /mnt/lvm/immich/photos:/usr/src/app/upload
    - ./immich/config:/usr/src/app/config
  environment:
    DB_HOSTNAME: immich-db
    DB_USERNAME: immich
    DB_PASSWORD: "[PLACEHOLDER]"
    DB_DATABASE: immich
    REDIS_HOSTNAME: immich-redis
  depends_on:
    - immich-db
    - immich-redis
  restart: unless-stopped
  networks:
    - media_net

immich-machine-learning:
  image: ghcr.io/immich-app/immich-machine-learning:release
  container_name: immich-machine-learning
  volumes:
    - ./immich/config:/usr/src/app/config
  environment:
    DB_HOSTNAME: immich-db
    DB_USERNAME: immich
    DB_PASSWORD: "[PLACEHOLDER]"
    DB_DATABASE: immich
    REDIS_HOSTNAME: immich-redis
  depends_on:
    - immich-db
    - immich-redis
  restart: unless-stopped
  networks:
    - media_net

immich-db:
  image: postgres:15-alpine
  container_name: immich-db
  volumes:
    - ./immich/db:/var/lib/postgresql/data
  environment:
    POSTGRES_DB: immich
    POSTGRES_USER: immich
    POSTGRES_PASSWORD: "[PLACEHOLDER]"
  restart: unless-stopped
  networks:
    - media_net

immich-redis:
  image: redis:7-alpine
  container_name: immich-redis
  restart: unless-stopped
  networks:
    - media_net
```

#### Matrix Synapse (with LiveKit)

```yaml
synapse:
  image: matrixdotorg/synapse:latest
  container_name: synapse
  ports:
    - "10.9.0.4:8008:8008"
  volumes:
    - ./synapse/data:/data
    - ./synapse/config:/etc/matrix-synapse
  environment:
    SYNAPSE_SERVER_NAME: matrix.aryadivap.com
    SYNAPSE_REPORT_STATS: "no"
    POSTGRES_HOST: synapse-db
    POSTGRES_DB: synapse
    POSTGRES_USER: synapse
    POSTGRES_PASSWORD: "[PLACEHOLDER]"
  depends_on:
    - synapse-db
  restart: unless-stopped
  networks:
    - media_net

synapse-db:
  image: postgres:15-alpine
  container_name: synapse-db
  volumes:
    - ./synapse/db:/var/lib/postgresql/data
  environment:
    POSTGRES_DB: synapse
    POSTGRES_USER: synapse
    POSTGRES_PASSWORD: "[PLACEHOLDER]"
  restart: unless-stopped
  networks:
    - media_net

matrix-rtc-jwt:
  image: ghcr.io/matrix-org/matrix-rtc-jwt:latest
  container_name: matrix-rtc-jwt
  ports:
    - "10.9.0.4:8081:8081"
  volumes:
    - ./synapse/rtc-jwt:/config
  restart: unless-stopped
  networks:
    - media_net

livekit-server:
  image: livekit/livekit-server:latest
  container_name: livekit-server
  ports:
    - "10.9.0.4:7880:7880"
    - "10.9.0.4:7881:7881"
    - "10.9.0.4:50100-50200:50100-50200/udp"
  volumes:
    - ./synapse/livekit:/etc/livekit
  restart: unless-stopped
  networks:
    - media_net
```

#### Jellyfin


```yaml
jellyfin:
  image: jellyfin/jellyfin:latest
  container_name: jellyfin
  ports:
    - "10.9.0.4:8096:8096"
  volumes:
    - ./jellyfin/config:/config
    - ./jellyfin/cache:/cache
    - /mnt/lvm/media:/media:ro
  devices:
    - /dev/dri:/dev/dri  # Optional: hardware transcoding
  restart: unless-stopped
  networks:
    - media_net
```

#### Prowlarr

```yaml
prowlarr:
  image: lscr.io/linuxserver/prowlarr:latest
  container_name: prowlarr
  ports:
    - "10.9.0.4:9696:9696"
  volumes:
    - ./prowlarr/config:/config
  environment:
    TZ: "Asia/Jakarta"
  restart: unless-stopped
  networks:
    - media_net
```

#### Radarr

```yaml
radarr:
  image: lscr.io/linuxserver/radarr:latest
  container_name: radarr
  ports:
    - "10.9.0.4:7878:7878"
  volumes:
    - ./radarr/config:/config
    - /mnt/lvm/media/movies:/movies
    - /mnt/lvm/downloads:/downloads
  environment:
    TZ: "Asia/Jakarta"
  restart: unless-stopped
  networks:
    - media_net
```

---

### Raspberry Pi — VPN-Isolated Network (`gluetun`)

#### Gluetun + qBittorrent

```yaml
gluetun:
  image: qmcgaw/gluetun:latest
  container_name: gluetun
  ports:
    - "10.9.0.4:8080:8080"  # qBittorrent Web UI
  volumes:
    - ./gluetun/config:/gluetun
  environment:
    VPN_SERVICE_PROVIDER: surfshark
    VPN_TYPE: wireguard
    WIREGUARD_PRIVATE_KEY: "[PLACEHOLDER]"
    WIREGUARD_ADDRESSES: "[PLACEHOLDER]"
    SERVER_REGIONS: "[PLACEHOLDER: e.g., Australia]"
  cap_add:
    - NET_ADMIN
  restart: unless-stopped

qbittorrent:
  image: lscr.io/linuxserver/qbittorrent:latest
  container_name: qbittorrent
  network_mode: "service:gluetun"
  volumes:
    - ./qbittorrent/config:/config
    - /mnt/lvm/downloads:/downloads
  environment:
    TZ: "Asia/Jakarta"
    UMASK: "002"
  depends_on:
    gluetun:
      condition: service_healthy
  restart: unless-stopped
```

---

### Cloud VPS — Services (`vps_net`)

#### Pi-hole

```yaml
pihole:
  image: pihole/pihole:latest
  container_name: pihole
  ports:
    - "10.9.0.1:53:53/tcp"
    - "10.9.0.1:53:53/udp"
    - "127.0.0.1:80:80/tcp"
  environment:
    TZ: "Asia/Jakarta"
    WEBPASSWORD: "[PLACEHOLDER: admin password]"
    PIHOLE_DNS_: "1.1.1.1;1.0.0.1"
    DNSSEC: "true"
    CONDITIONAL_FORWARDING: "false"
  volumes:
    - ./pihole/config:/etc/pihole
    - ./pihole/dnsmasq:/etc/dnsmasq.d
  restart: unless-stopped
  networks:
    vps_net:
      ipv4_address: "10.9.0.1"
```

#### Dashy

```yaml
dashy:
  image: lissy93/dashy:latest
  container_name: dashy
  ports:
    - "10.9.0.1:8080:80"
  volumes:
    - ./dashy/conf.yml:/app/public/conf.yml
    - ./dashy/icons:/app/public/item-icons
  restart: unless-stopped
  networks:
    - vps_net
```

#### Vault Warden

```yaml
vaultwarden:
  image: vaultwarden/server:latest
  container_name: vaultwarden
  ports:
    - "10.9.0.1:8989:80"
  volumes:
    - ./vaultwarden/data:/data
  environment:
    SIGNUPS_ALLOWED: "false"    # Disable after creating accounts
    ADMIN_TOKEN: "[PLACEHOLDER: strong admin token]"
  restart: unless-stopped
  networks:
    - vps_net
```

#### Portainer (VPS instance)

```yaml
portainer-vps:
  image: portainer/portainer-ce:latest
  container_name: portainer-vps
  ports:
    - "10.9.0.1:9000:9000"
  volumes:
    - /var/run/docker.sock:/var/run/docker.sock:ro
    - ./portainer-vps/data:/data
  restart: unless-stopped
  networks:
    - vps_net
```

#### Uptime Kuma

```yaml
uptime-kuma-vps:
  image: louislam/uptime-kuma:latest
  container_name: uptime-kuma-vps
  ports:
    - "10.9.0.1:3001:3001"
  volumes:
    - ./uptime-kuma-vps/data:/app/data
  restart: unless-stopped
  networks:
    - vps_net
```

#### Caddy

```yaml
caddy:
  image: caddy:latest
  container_name: caddy
  ports:
    - "0.0.0.0:80:80"
    - "0.0.0.0:443:443"
  volumes:
    - ./caddy/Caddyfile:/etc/caddy/Caddyfile:ro
    - ./caddy/data:/data
    - ./caddy/config:/config
  restart: unless-stopped
  networks:
    - vps_net
```

**Caddyfile example:**

```caddy
jellyfin.aryadivap.com {
    reverse_proxy http://10.9.0.4:8096
}

nextcloud.aryadivap.com {
    reverse_proxy http://10.9.0.4:8080
}

immich.aryadivap.com {
    reverse_proxy http://10.9.0.4:2283
}
```

---

## Custom Networks Definition

### Raspberry Pi (`docker-compose.yml`)

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
```

> The `gluetun` service handles its own networking. qBittorrent uses `network_mode: "service:gluetun"` and does not need a separate network entry.

### Cloud VPS (`docker-compose.yml`)

```yaml
networks:
  vps_net:
    driver: bridge
    ipam:
      config:
        - subnet: "172.22.0.0/16"
```
