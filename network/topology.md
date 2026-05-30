# Network Topology

## Physical Topology

### WAN
- **Gateway:** TP-Link Omada ER605 Gigabit VPN Router
- **WAN Configuration:** Dual-ISP load balancing. ISP A is the primary active connection; ISP B is configured as backup/failover. Traffic is distributed across both links by usage, with automatic failover if one ISP goes down.
- **ISPs:** [PLACEHOLDER: provider names and speeds]

### LAN
- **Switching:** Managed Layer 2/3 switch fabric
- **Primary Subnet:** `10.9.0.0/24`
- **DHCP:** N/A

### Hosts
| Host | Role | Connection |
|------|------|------------|
| Raspberry Pi 5 | On-premises compute + storage | Wired to LAN switch |
| Cloud VPS | Public edge router, WireGuard server, Caddy reverse proxy, Pi-hole DNS | Internet-facing via provider |
| Garuda Arch Linux Desktop | GPU compute (Ollama/Open WebUI) | Wired to LAN switch |

---

## Overlay Network

### Primary Tunnel — WireGuard (Server / Client)
- **Architecture:** VPS acts as the **WireGuard server**. All LAN devices (Pi, Garuda Desktop, laptops) are **WireGuard clients** that connect to the VPS.
- **VPS Role:** WireGuard server routes traffic between peers, enabling LAN devices to communicate securely through the encrypted overlay.
- **Subnet:** `10.9.0.0/24`
- **VPS Listen Port:** 58120

### Fallback Tunnel — Tailscale
- **Type:** CGNAT-traversing mesh overlay
- **Use Cases:** Out-of-band administration, secondary access path, direct peer-to-peer connections between devices behind NAT
- **Subnet:** `100.x.y.z/32` (Tailscale-assigned)

---

## Reverse Proxy (Caddy)

- **Hosted on:** Cloud VPS
- **Role:** Terminates public HTTPS requests (subdomains pointing to VPS IP) and proxies them through the WireGuard tunnel to the appropriate service on the Raspberry Pi.
- **Domains:** Each subdomain (e.g., `jellyfin.example.com`) has a Cloudflare A record pointing to the VPS public IP. Caddy on the VPS matches the domain and proxies to `http://10.9.0.x:PORT` on the Pi.

---

## DNS (Pi-hole)

- **Hosted on:** Cloud VPS as a Docker container
- **DNS for LAN:** All LAN devices and the Raspberry Pi are configured to use the VPS WireGuard IP as their DNS server. Queries are forwarded through the WireGuard tunnel to the VPS, where Pi-hole resolves and filters them.
- **Upstream DNS:** Cloudflare 1.1.1.1 & Quad9 9.9.9.9

---

## Traffic Flow

### External Access (Internet → Service)

```
User
  │
  │ DNS: jellyfin.example.com → Cloudflare A record → VPS Public IP
  ▼
Cloud VPS
  │
  ├── Caddy reverse proxy (matches domain)
  │
  └── WireGuard tunnel → 10.9.0.x:PORT
         │
         ▼
      Raspberry Pi 5 (Docker container)
```

### Internal DNS Flow (LAN Device → Internet)

```
LAN Device
  │
  │ DNS = VPS WireGuard IP (10.9.0.1)
  │
  └── WireGuard tunnel
         │
         ▼
      Cloud VPS → Pi-hole container → Upstream DNS
```

### LAN Peer Communication

```
LAN Device (WireGuard client)
  │
  │ WG tunnel → VPS (server)
  │ VPS routes traffic to other peers
  │
  └── Raspberry Pi 5 (WireGuard client)
         │
         └── Docker services
```

---

## Reference Diagram

See [`../diagrams/Updated Homelab Architecture v3.drawio.png`](../diagrams/Updated%20Homelab%20Architecture%20v3.drawio.png) for the visual network topology.
