## Physical Topology

### WAN
- **Gateway:** TP-Link Omada ER605 Gigabit VPN Router
- **WAN Configuration:** Dual-ISP load balancing with automatic failover. ISP A (**Ovall**) is bound as the primary active connection; ISP B (**Bnet**) is configured as backup/failover. Traffic is distributed across both links by usage, with automatic failover if one ISP goes down.
- **ISPs:** Ovall (primary), Bnet (backup)
- **Note:** The ER605 is fully isolated from WireGuard overlay networks. It handles only local WAN/LAN routing — no port forwarding to VPS or WG subnets.
### LAN
- **Switching:** Managed Layer 2/3 switch fabric
- **Primary Subnet:** `192.168.10.0/24` (physical LAN)
- **DHCP:** ER605 handles DHCP for the physical LAN
### Hosts
| Host | Role | Connection |
|------|------|------------|
| Raspberry Pi 5 | On-premises compute + storage | Wired to LAN switch |
| Cloud VPS | Public edge router, WireGuard server, Caddy reverse proxy, Pi-hole DNS | Internet-facing via provider |
| Garuda Arch Linux Desktop | GPU compute (Ollama/Open WebUI) | Wired to LAN switch |

---

## Overlay Network

### Primary Tunnel — WireGuard (Server / Client)
- **Architecture:** VPS acts as the **WireGuard server**. All LAN devices (Pi, Garuda Desktop, laptops, phones) are **WireGuard clients** that connect to the VPS.
- **VPS Role:** WireGuard server routes traffic between peers, enabling LAN devices to communicate securely through the encrypted overlay.
- **Subnet:** `10.9.0.0/24`
- **VPS Listen Port:** `51820`/UDP

### Fallback Tunnel — Tailscale
- **Type:** CGNAT-traversing mesh overlay
- **Use Cases:** Out-of-band administration, secondary access path, direct peer-to-peer connections between devices behind NAT
- **Subnet:** `100.x.y.z/32` (Tailscale-assigned)

---

## Reverse Proxy (Caddy)

- **Hosted on:** Cloud VPS
- **Role:** Terminates public HTTPS requests (subdomains pointing to VPS IP) and proxies them through the WireGuard tunnel to the appropriate service on the Raspberry Pi.
- **Domains:** Each subdomain (e.g., `jellyfin.aryadivap.com`) has a Cloudflare A record pointing to the VPS public IP. Caddy on the VPS matches the domain and proxies to `http://10.9.0.x:PORT` on the Pi.
- **VPS-hosted services (Dashy, Vault Warden):** Proxied locally on the VPS without traversing the tunnel.

---

## DNS (Pi-hole)

- **Hosted on:** Cloud VPS as a Docker container
- **DNS for LAN:** All LAN devices and the Raspberry Pi are configured to use the VPS WireGuard IP (`10.9.0.1`) as their DNS server. Queries are forwarded through the WireGuard tunnel to the VPS, where Pi-hole resolves and filters them.
- **Upstream DNS:** Cloudflare `1.1.1.1` & Quad9 `9.9.9.9`

---

## Traffic Flow

### External Access (Internet → Service)


