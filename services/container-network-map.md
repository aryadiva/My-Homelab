# Container Network Map

## Network Isolation Model

Two Docker hosts exist in this infrastructure, connected via the WireGuard overlay:

### Raspberry Pi 5 — Docker Networks

```
┌─────────────────────────────────────────────────┐
│             Raspberry Pi 5 (Host)                │
│                                                  │
│  ┌──────────────┐    ┌──────────────────────────────┐   │
│  │ infra_net     │    │ media_net                      │   │
│  │ (bridge)      │    │ (bridge)                       │   │
│  │               │    │                                │   │
│  │  (*)Homepage  │    │  Nextcloud (+DB,R)             │   │
│  │  Portainer    │    │  Immich (+DB,R,ML)             │   │
│  │               │    │  Matrix Synapse (+PG15)        │   │
│  │               │    │  LiveKit Server (+RTC JWT)     │   │
│  │               │    │  Jellyfin                      │   │
│  │               │    │  Prowlarr                      │   │
│  │               │    │  Radarr                        │   │
│  └──────────────┘    └──────────────────────────────┘   │
│                                                  │
│  ┌──────────────────────────────────────────┐    │
│  │  gluetun (container, NET_ADMIN)          │    │
│  │  ┌────────────────────────────────────┐  │    │
│  │  │  qBittorrent (network_mode:        │  │    │
│  │  │    service:gluetun)                │  │    │
│  │  └────────────────────────────────────┘  │    │
│  └──────────────────────────────────────────┘    │
│                                                  │
│  ┌──────────────────────────────────────────┐    │
│  │  WireGuard client (wg0)                  │    │
│  │  Connects to VPS WireGuard server        │    │
│  └──────────────────────────────────────────┘    │
└─────────────────────────────────────────────────┘
(*) Homepage on Pi is deprecated — Dashy on VPS is now the primary dashboard
```

> **Note:** The primary dashboard has migrated from Homepage (Pi, `infra_net`) to Dashy (VPS, `vps_net`).

### Cloud VPS — Docker Networks

```
┌─────────────────────────────────────────────────┐
│              Cloud VPS (Host)                     │
│                                                   │
│  ┌──────────────────────────────────────┐        │
│  │  vps_net (bridge)                     │        │
│  │                                       │        │
│  │  Pi-hole (DNS, 10.9.0.1:53)           │        │
│  │  Dashy (Dashboard, 10.9.0.1:8080)      │        │
│  │  Vault Warden (Password Mgr, 10.9.0.1:8989)   │
│  │  Uptime Kuma (Monitor, 10.9.0.1:3001)         │
│  │  Portainer (VPS instance, 10.9.0.1:9000)     │
│  └──────────────────────────────────────┘        │
│                                                   │
│  ┌──────────────────────────────────────────┐    │
│  │  WireGuard server (wg0, native)          │    │
│  │  Listens on 10.9.0.1:51820/UDP          │    │
│  │  Routes traffic between peers            │    │
│  │  Forwards DNS → Pi-hole container        │    │
│  └──────────────────────────────────────────┘    │
└─────────────────────────────────────────────────┘
```

---

## Port Mapping Summary

### Raspberry Pi 5

| Service | Host IP | Host Port | Container Port | Protocol | Network |
|---------|---------|-----------|----------------|----------|---------|
| Portainer (Pi) | `10.9.0.4` | 9000 | 9000 | TCP | `infra_net` |
| Homepage | `10.9.0.4` | 3000 | 3000 | TCP | `infra_net` |
| Matrix Synapse | `10.9.0.4` | 8008 | 8008 | TCP | `media_net` |
| Matrix RTC JWT | `10.9.0.4` | 8081 | 8081 | TCP | `media_net` |
| LiveKit (Signal) | `10.9.0.4` | 7880 | 7880 | TCP | `media_net` |
| LiveKit (TURN/TCP) | `10.9.0.4` | 7881 | 7881 | TCP | `media_net` |
| LiveKit (UDP media) | `10.9.0.4` | 50100-50200 | 50100-50200 | UDP | `media_net` |
| Nextcloud | `10.9.0.4` | 8080 | 80 | TCP | `media_net` |
| Immich | `10.9.0.4` | 2283 | 2283 | TCP | `media_net` |
| Jellyfin | `10.9.0.4` | 8096 | 8096 | TCP | `media_net` |
| Prowlarr | `10.9.0.4` | 9696 | 9696 | TCP | `media_net` |
| Radarr | `10.9.0.4` | 7878 | 7878 | TCP | `media_net` |
| qBittorrent Web UI | `10.9.0.4` | 8080 | 8080 | TCP | via Gluetun |
| Gluetun | N/A | N/A | N/A | N/A | host/NET_ADMIN |

### Cloud VPS

| Service | Host IP | Host Port | Container Port | Protocol | Network |
|---------|---------|-----------|----------------|----------|---------|

| Pi-hole (DNS) | `10.9.0.1` | 53 | 53 | TCP+UDP | `vps_net` |
| Pi-hole (Admin) | `127.0.0.1` | 80 | 80 | TCP | `vps_net` |
| Caddy (HTTP) | `0.0.0.0` | 80 | 80 | TCP | `vps_net` |
| Caddy (HTTPS) | `0.0.0.0` | 443 | 443 | TCP | `vps_net` |
| Dashy | `10.9.0.1` | 8080 | 80 | TCP | `vps_net` |
| Vault Warden | `10.9.0.1` | 8989 | 80 | TCP | `vps_net` |
| Portainer (VPS) | `10.9.0.1` | 9000 | 9000 | TCP | `vps_net` |
| Uptime Kuma | `10.9.0.1` | 3001 | 3001 | TCP | `vps_net` |

---

## Communication Rules

| From | To | Allowed | Notes |
|------|----|---------|-------|
| `infra_net` (Pi) | Host network (Pi) | Ports mapped | Exposed via host port binding |
| `media_net` (Pi) | Host network (Pi) | Ports mapped | Exposed via host port binding |
| `infra_net` | `media_net` | **No** | Separate bridge networks |
| `gluetun` | Internet | VPN tunnel (Surfshark) | All egress via WireGuard/OpenVPN |
| `gluetun` | Pi WireGuard IP | Ports specified | qBittorrent Web UI accessible over WG |
| Pi services | VPS (via WireGuard) | Specific ports | Caddy proxies: `10.9.0.x:PORT` |
| VPS (Pi-hole) | WireGuard clients (peers) | `53/UDP`, `53/TCP` | DNS for all LAN devices over WG |
| LAN devices | VPS WireGuard IP | `53/UDP` | DNS queries forwarded through WG tunnel |
| VPS services (Dashy, Vault Warden, Uptime Kuma, Portainer) | VPS local | VPS internal ports | Same host — no tunnel needed |

---

## Traffic Flow: External Request

```
User → cloudflare.example.com (A record → VPS public IP)
  │
  ▼
VPS (port 443) → Caddy container
  │
  │ Matches domain → reverse proxy to Pi via WireGuard
  ▼
WireGuard tunnel → 10.9.0.x:PORT
  │
Pi Docker container (media_net / infra_net)
```

## Traffic Flow: Internal DNS

```

LAN device → DNS = 10.9.0.1 (VPS WireGuard IP)
  │
  │ WireGuard tunnel
  ▼
VPS WireGuard server → Pi-hole container (vps_net)
  │
  │ Query resolved or forwarded
  ▼
Upstream DNS server
```
