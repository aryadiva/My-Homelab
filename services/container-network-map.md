# Container Network Map — Isolation Model (Synced 2026-11-16)

> This document describes Docker network isolation across hosts and services.  
> Values marked "historical" should be verified on live systems where possible via `docker network inspect`.

---

## Raspberry Pi 5 — Network Zones

```
┌─────────────────────────────────────────┐
│ Raspberry Pi 5 (Docker Host)            │
├─────────────────────────────────────────┤
│ ┌──────────────────────────────────────┐│
│ │   infra_net                         ││
│ │   bridge network:                    ││
│ │   • Homepage (deprecated)            ││
│ │   • Portainer Server                 ││
│ │   • Network isolation from media     ││
│ └──────────────────────────────────────┘│
│                                          │
│ ┌──────────────────────────────────────┐│
│ │   media_net                         ││
│ │   bridge network:                    ││
│ │   • Nextcloud (+MariaDB, Redis)      ││
│ │   • Immich (server + ML containers)  ││
│ │   • Jellyfin                        ││
│ │   • Prowlarr / Radarr               ││
│ │   • Matrix Synapse (+Postgres DB)    ││
│ │   • LiveKit Server                  ││
│ └──────────────────────────────────────┘│
│                                          │
│ ┌──────────────────────────────────────┐│
│ │   gluetun                            ││
│ │   (host namespace, NET_ADMIN)        ││
│ │   └─ qBittorrent (shares namespace)  ││
│ └──────────────────────────────────────┘│
│                                          │
│ ┌──────────────────────────────────────┐│
│ │   WireGuard client (wg0)             ││
│ │   Connects to VPS peer               ││
│ └──────────────────────────────────────┘│
└─────────────────────────────────────────┘
Notes:
• Infra and media are isolated; no cross-communication possible without explicit routing.
• gluetun has NET_ADMIN capability — required for VPN tunnel creation.
```

> Homepage is deprecated — Dashy on VPS now serves as central portal. Keep only if legacy workflows depend on it.


## Cloud VPS — Network Map

```
┌───────────────────────────────────────────┐
│      Cloud VPS (Docker Host)              │
├───────────────────────────────────────────┤
│ ┌────────────────────────────────────────┐│
│ │   vps_net                              ││
│ │   bridge network:                      ││
│ │   • Pi-hole (10.9.0.1:53 UDP/TCP)      ││
│ │   • Dashy                              ││
│ │   • Vaultwarden                        ││
│ │   • Uptime Kuma                        ││
│ │   • Caddy (public TLS termination)*    ││
│ └────────────────────────────────────────┘│
│                                           │
│ ┌────────────────────────────────────────┐│
│ │   WireGuard server (wg0, host-native)  ││
│ │   Listen: 51820/udp                    ││
│ │   IP: 10.9.0.1/24                      ││
│ │                                         ││
│ │   Routes traffic between all peers     ││
│ │   (Pi, Orange Pi, Garuda desktop,      ││
│ │    personal laptops, phones)           ││
│ └────────────────────────────────────────┘│
└───────────────────────────────────────────┘

Notes:
• WireGuard runs natively on OS — not in Docker compose container.
• Caddy terminates HTTPS publicly and proxies requests back to hosts via WG tunnel.
• Pi-hole resolves DNS for all peers over the overlay network.
```

## Port Binding Summary

### Raspberry Pi 5 Exports

| Service         | Host IP        | Port     | Target Container       | Network   |
|-----------------|----------------|----------|------------------------|-----------|
| Nextcloud       | 10.9.0.4       | :8080    | nextcloud:80           | media_net |
| Immich          | 10.9.0.4       | :2283    | immich-server:2283     | media_net |
| Jellyfin        | 10.9.0.4       | :8096    | jellyfin:8096          | media_net |
| Matrix Synapse  | 10.9.0.4       | :8008    | synapse:8008           | media_net |
| LiveKit (TCP)   | 10.9.0.4       | :7880/7881| livekit-server         | media_net |
| Radarr          | 10.9.0.4       | :7878    | radarr:7878            | media_net |

### Cloud VPS Exports

| Service   | Host IP        | Port     | Notes                              |
|-----------|----------------|----------|------------------------------------|
| Caddy     | 0.0.0.0        | :443/80  | Public HTTPS entry point           |
| Dashy     | 10.9.0.1       | :8080    | Same port as Nextcloud on Pi (via WG) |
| Vaultwarden| 10.9.0.1      | :8989    | Password server                     |
| Uptime Kuma| 10.9.0.1      | :3001    | Monitoring                          |

---

## Communication Patterns

### External Access → Internal Service

```
User (internet)
  └── Caddy on VPS (port 443, TLS)
       └── Proxy over WireGuard
            └── Raspberry Pi service (host_ip:port)
                 └── Docker container
```

### LAN/Internal DNS Resolution

```
Device (DNS set to 10.9.0.1)
  └── DNS query over WireGuard
       └── VPS -> Pi-hole container
            └── Ad block filter → resolved IP
                 └── Returns local or external IP
```

### Peering Between Containers (Same Network)

- `media_net` containers communicate directly via Docker overlay.
  Example: Nextcloud uses internal Redis (`nextcloud-redis`) without host exposure.
- `infra_net` — isolated; only host-mapped ports reachable externally.

### qBittorrent + Gluetun Isolation

- qBittorrent binds to `network_mode: service:gluetun`, sharing Gluetun's network namespace.
- All egress from qBittorrent tunnels via Surfshark WireGuard VPN (not local WG).
```
qBittorrent → gluetun share namespace → VPN tunnel → internet exit node
                 no direct host routing for torrent peers
```

---
