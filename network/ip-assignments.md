# IP Assignments

## LAN Subnet: `10.9.0.0/24` / `192.168.10.0/24`

| Device | Interface | IP Address | Notes |
|--------|-----------|------------|-------|
| ER605 Router | LAN | `192.168.10.1` | Default gateway, DHCP server |
| Raspberry Pi 5 | eth0 | `192.168.10.188` | Static reservation |
| Raspberry Pi 5 | WireGuard (wg0) | `10.9.0.4` | WireGuard client (peer) |
| Cloud VPS | WireGuard (wg0) | `10.9.0.1` | WireGuard server |
| Garuda Arch Desktop | eth0 | `10.9.0.2` / `192.168.10.201` | Static reservation |
| LAN devices (laptops, phones) | WiFi | `10.9.0.x` | WireGuard clients connecting to VPS |
| [PLACEHOLDER: other reserved IPs] | | | |

## WireGuard Server Subnet: `10.9.0.0/24`

### VPS (WireGuard Server)

| Interface | IP | Listen Port |
|-----------|----|-------------|
| wg0 | `10.9.0.1` | 51820 |

### WireGuard Clients (Peers)

| Client | WireGuard IP | Endpoint | Allowed IPs | Notes |
|--------|-------------|----------|-------------|-------|
| Raspberry Pi 5 | `10.9.0.4` | VPS public IP | `10.9.0.0/24` | Primary on-prem node |
| Garuda Arch Desktop | `10.9.0.2` | VPS public IP | `10.9.0.0/24` | GPU compute node |
| [other clients] | `10.9.0.x` | VPS public IP | `10.9.0.0/24` | Laptops, phones, etc. |

## Tailscale

| Device | Tailscale IP | Machine Name |
|--------|-------------|--------------|
| Raspberry Pi 5 | `100.121.73.43` | arya-ubuntu |
| Cloud VPS | `N/A` | `N/A` |
| Garuda Arch Desktop | `100.70.202.90` | arya-garuda |

## DNS

- **Primary DNS:** Pi-hole (resolved via VPS WireGuard IP `10.9.0.1`)
- **Upstream DNS:** Cloudflare `1.1.1.1` & Quad9 `9.9.9.9`
- **Local Domain:** `aryadivap.com` -> main domain used for CV/Portfolio dpeloyed with Vercel
- All LAN devices and the Raspberry Pi point DNS to `10.9.0.1` (VPS WireGuard IP) → forwarded through WireGuard tunnel → Pi-hole container on VPS

## Caddy Reverse Proxy

| Subdomain | Target (via WireGuard) | Notes |
|-----------|-----------------------|-------|
| `nextcloud.aryadivap.com` | `http://10.9.0.4:80` | `Nextcloud` |
| `immich.aryadivap.com` | `http://10.9.0.4:2283` | `Immich` |
| `jellyfin.aryadivap.com` | `http://10.9.0.4:8096` | `Jellyfin` |
| `portainer.aryadivap.com` | `http://10.9.0.1:9000` | `Portainer-vps` |
| `portainer-home.aryadivap.com` | `https://10.9.0.4:9000` | `Portainer-Home` |
| `pihole.aryadivap.com` | `https://10.9.0.1:53` | `PiHole` |
| `ai.aryadivap.com` | `https://10.9.0.4:3000` | `OpenWebUI` |
| `home.aryadivap.com` | `https://10.9.0.1:8080` | `Dashy` |
| `vault.aryadivap.com` | `https://10.9.0.1:8989` | `Vault Warden` |
| `uptime.aryadivap.com` | `https://10.9.0.1:3001` | `Uptime Kuma` |
| `matrix.aryadivap.com` | `http://10.9.0.4:8008` | `Matrix Synapse (client-server API)` |
| `element.aryadivap.com` | `http://10.9.0.4:8008` | `Element Web client (via Caddy subpath)` |

