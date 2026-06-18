# IP Assignments

## Physical LAN Subnet: `192.168.10.0/24`
> The physical LAN is managed by the ER605. All devices on the local network are assigned IPs via ER605 DHCP. The WireGuard overlay subnet (`10.9.0.0/24`) is a separate encrypted tunnel — devices communicate between the two subnets through the VPS as the WireGuard server.
| Device | Interface | IP Address | Notes |
|--------|-----------|------------|-------|
| ER605 Router | LAN | `192.168.10.1` | Default gateway, DHCP server |
| Raspberry Pi 5 | eth0 | `192.168.10.188` | Static reservation |
| Garuda Arch Desktop | eth0 | `192.168.10.201` | Static reservation |

---
## WireGuard Server Subnet: `10.9.0.0/24`

This is the **overlay tunnel** — all LAN traffic between devices and services flows through this encrypted WireGuard tunnel.

### VPS (WireGuard Server)

| Interface | IP | Listen Port |
|-----------|----|-------------|
| wg0 | `10.9.0.1` | `51820`/UDP |

### WireGuard Clients (Peers)

| Client | WireGuard IP | Endpoint | Allowed IPs | Notes |
|--------|-------------|----------|-------------|-------|
| Raspberry Pi 5 | `10.9.0.4` | VPS public IP | `10.9.0.0/24` | Primary on-prem node |
| Garuda Arch Desktop | `10.9.0.2` | VPS public IP | `10.9.0.0/24, 192.168.10.0/24` | GPU compute node |
| Windows Laptop | `10.9.0.3` | VPS public IP | `10.9.0.0/24` | Remote client |
| Samsung Phone | `10.9.0.5` | VPS public IP | `10.9.0.0/24` | Mobile device |
| OrangePi | `10.9.0.6` | VPS public IP | `10.9.0.0/24` | Additional compute node |

## Tailscale

| Device | Tailscale IP | Machine Name |
|--------|-------------|--------------|
| Raspberry Pi 5 | `100.121.73.43` | arya-ubuntu |
| Cloud VPS | N/A | N/A |
| Garuda Arch Desktop | `100.70.202.90` | arya-garuda |

## DNS

- **Primary DNS:** Pi-hole (resolved via VPS WireGuard IP `10.9.0.1`)
- **Upstream DNS:** Cloudflare `1.1.1.1` & Quad9 `9.9.9.9`
- **Local Domain:** `aryadivap.com` — main domain used for CV/Portfolio deployed with Vercel
- All LAN devices and the Raspberry Pi point DNS to `10.9.0.1` (VPS WireGuard IP) → forwarded through WireGuard tunnel → Pi-hole container on VPS

## Caddy Reverse Proxy

All subdomains point to the VPS public IP. Caddy on the VPS matches the domain and proxies traffic:
| Subdomain | Target (via WireGuard) | Host | Notes |
|-----------|-----------------------|------|-------|
| `nextcloud.aryadivap.com` | `http://10.9.0.4:8080` | Pi | Nextcloud |
| `immich.aryadivap.com` | `http://10.9.0.4:2283` | Pi | Immich |
| `jellyfin.aryadivap.com` | `http://10.9.0.4:8096` | Pi | Jellyfin |
| `portainer.aryadivap.com` | `http://10.9.0.1:9000` | VPS | Portainer (VPS instance) |
| `portainer-home.aryadivap.com` | `https://10.9.0.4:9000` | Pi | Portainer (Pi instance) |
| `pihole.aryadivap.com` | `http://127.0.0.1:80` | VPS | Pi-hole admin (local only) |
| `home.aryadivap.com` | `https://10.9.0.1:8080` | VPS | Dashy |
| `vault.aryadivap.com` | `https://10.9.0.1:8989` | VPS | Vault Warden |
| `uptime.aryadivap.com` | `https://10.9.0.1:3001` | VPS | Uptime Kuma |
| `ai.aryadivap.com` | `http://10.9.0.2:3000` | Garuda | Open WebUI (via Ollama GPU) |
| `matrix.aryadivap.com` | `http://10.9.0.4:8008` | Pi | Matrix Synapse (client-server API) |
| `element.aryadivap.com` | `http://10.9.0.4:8008` | Pi | Element Web client (via Caddy subpath)

