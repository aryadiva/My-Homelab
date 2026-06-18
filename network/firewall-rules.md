# Firewall Rules

## ER605 (Hardware Gateway) — Load Balancing & Failover

The ER605 is configured for **dual-ISP load balancing with automatic failover**. No inbound port forwarding is configured at the ER605 level — all external access is handled through the cloud VPS. The ER605 is fully isolated from WireGuard overlay networks and only handles physical WAN/LAN routing.

### WAN Policy
| ISP | Role | Weight | Notes |
|-----|------|--------|-------|
| **Ovall** (ISP A) | Primary (active) | 1 | Handles the majority of traffic |
| **Bnet** (ISP B) | Backup/failover | 1 | Takes over if ISP A goes down |

### Load Balancing Mode
- **Algorithm:** Online Backup
- **Failover Detection:** Ping to `8.8.8.8` and `1.1.1.1` on both WANs

### WAN Rules
| Rule # | Direction | Source | Destination | Port/Protocol | Action | Notes |
|--------|-----------|--------|-------------|---------------|--------|-------|
| 1 | Inbound | Any | ER605 WAN | ICMP | Allow | Ping for monitoring |
| 2 | Inbound | Any | Any | All | Deny | Default deny all inbound — no port forwards |
| 3 | Outbound | Any | ER605 WAN | All | Allow | For connection |

> **Note:** There are **no port forwarding rules** on the ER605. All public ingress goes through the VPS.

### LAN Rules
| Rule # | Direction | Source | Destination | Port/Protocol | Action | Notes |
|--------|-----------|--------|-------------|---------------|-------|-------|
| 1 | Inbound | LAN | Any | All | Allow | Default LAN access |

---

## Cloud VPS Firewall

| Rule | Source | Destination | Port | Protocol | Notes |
|------|--------|-------------|------|----------|-------|
| 1 | Any | VPS | `51820` | UDP | WireGuard listen port |
| 2 | Any | VPS | `443` | TCP | HTTPS — Caddy reverse proxy |
| 3 | Any | VPS | `80` | TCP | HTTP — Caddy redirect to HTTPS |
| 4 | WireGuard subnet (`10.9.0.0/24`) | VPS internal | All | Any | Allow tunnel traffic |
| 5 | Any | VPS | All | Any | Deny all other inbound |

---

## Pi-hole (DNS Sinkhole — hosted on VPS)

### Blocklists
- Active Blocklists: 
    - StevenBlack
    - OISD
    - Hagezi
    - urlhaus.abuse.ch

### Local DNS Rewrites
| Domain | Target IP | Notes |
|--------|-----------|-------|
| `pihole.aryadivap.com` | `10.9.0.1` | Local service resolution over WireGuard |
| `Portainer` | `10.9.0.1` | VPS docker management |
| `portainer-home` | `10.9.0.4` | Pi docker management |

### DHCP
- Use static IP (DHCP disabled for LAN).
- Pi-hole is used exclusively for DNS filtering.

---

## Container-Level Firewall / Network Isolation (Raspberry Pi 5)

### Gluetun VPN Namespace
- **Policy:** qBittorrent is bound to Gluetun's network namespace — no direct host network access
- **Kill Switch:** Enabled (traffic drops if VPN disconnects)
- **Allowed Subnets:** `10.9.0.0/24` (LAN + WireGuard access for Web UI)

### Docker Bridge Networks
- Containers on the same Docker bridge network can communicate freely.
- Cross-network communication is restricted unless explicitly routed.
- See [`../services/container-network-map.md`](../services/container-network-map.md) for detailed network isolation per container.

---

> **Note:** Actual rule IDs and specific port numbers are replaced with placeholders for security. Adjust based on your ER605 and VPS firewall configuration.

