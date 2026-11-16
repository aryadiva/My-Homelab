# Firewall Rules — Synced 2026-11-16

> Values from historical docs and KB; verify against live state on VPS before making changes.

---

## ER605 Router (Hardware Gateway)

**Role:** Dual-ISP load balancing, failover, DNS upstream forwarding  
**Note:** ER605 is fully isolated from WireGuard overlay network  
CGNAT applies — no inbound port forwarding on the home network itself; all external access routes through VPS.

### WAN Policy

| Rule # | Source  | Destination | Port   | Protocol | Action     | Notes                          |
|--------|---------|-------------|--------|----------|------------|--------------------------------|
| 1      | Any     | ER605 WAN   | \-     | ICMP     | Allow      | Ping for monitoring            |
| 2      | Any     | `0.0.0.0/0` | All    | TCP+UDP  | Deny       | Default deny inbound           |
| 3      | Any     | ER605 WAN   | All    | TCP+UDP  | Allow      | Outbound traffic               |

### Load Balancing / Failover

- **ISP A (Ovall):** Primary active link (confirmed by old docs)
- **ISP B (Bnet):** Backup/failover when ISP A down
- **Algorithm:** Online backup with ping to `8.8.8.8` + `1.1.1.1` on both ISPs

> ⚠️ This configuration exists but is not verified in the current KB snapshot. Verify before changes.

---

## Cloud VPS — UFW Configuration

**Default policies:**
- Incoming: Deny  
- Outgoing: Allow  
- Forwarded/routed traffic: Allow  

### Allowed Rules Summary

| Source IP / Network | Destination | Port   | Protocol | Notes                                   |
|---------------------|-------------|--------|----------|-----------------------------------------|
| Any                 | VPS         | 51820  | UDP      | WireGuard listen port                   |
| Any                 | VPS         | 443    | TCP+UDP  | HTTPS and HTTP/3 (QUIC) for Caddy       |
| Any                 | VPS         | 80     | TCP      | HTTP redirect to HTTPS                  |
| `10.9.0.0/24`       | Any         | All    | All      | WireGuard internal overlay traffic      |
| Any                 | Any         | All    | All      | **Not allowed**                         |

### SSH Rate Limiting

- Port `22/tcp`: `LIMIT` action for rate-limiting (UFW built-in protection)

---

## Docker-Level Firewall & Isolation

### VPS Docker Services Binding

Services on the VPS (Caddy, Dashy, Vaultwarden, Portainer, Uptime Kuma) bind to **WireGuard IP (`10.9.0.1`)** instead of `0.0.0.0`. Reduces exposure surface per security guidance in KB.

### Raspberry Pi 5 Docker Bridge Networks

| Network Name    | Purpose              | Notes                                  |
|-----------------|----------------------|----------------------------------------|
| `infra_net`     | Infrastructure       | Portainer Server, deprecated Homepage  |
| `media_net`     | Media / Data services| Nextcloud, Immich, Matrix, Jellyfin    |
| `gluetun`       | VPN isolated         | qBittorrent (shares namespace with Gluetun container) |

- Containers on the same bridge network can communicate freely.
- Cross-network communication is restricted unless explicitly routed.

## Pi-hole (on VPS) — DNS Sinkhole Policy (Reference)

### Active Blocklists (Historical)

- StevenBlack  
- OISD  
- Hagezi  
- urlhaus.abuse.ch  

### Local Resolution Example

| Domain              | Rewritten To    | Purpose                      |
|---------------------|-----------------|------------------------------|
| `pihole.aryadivap.com` | VPS 127.0.0.1:80 | Pi-hole admin page proxy     |

> Blocklist and local resolution rules are for reference — verify with Pi-hole dashboard (status → blocklists) and DNS testing on LAN devices.

---

## Container-Level Firewall Policies (qBittorrent + Gluetun)

| Policy                               | Rule                                           | Notes                                    |
|--------------------------------------|------------------------------------------------|------------------------------------------|
| qBittorrent network namespace        | Share with Gluetun (`network_mode: service:gluetun`) | Inherits Gluetun's VPN tunnel            |
| Kill switch                          | Enabled                                         | Drops traffic if VPN disconnects         |
| Allowed subnets on gluetun peer      | `10.9.0.0/24`                                   | Web UI accessible over WireGuard         |

See [`services/container-network-map.md`](container-network-map.md) for port maps.

---

## Security Services on VPS (Reference)

| Service       | Function                            | Notes                                       |
|---------------|-------------------------------------|---------------------------------------------|
| CrowdSec      | Log acquisition + bouncer           | Previously logged connection refused errors. Verify current integration with `fail2ban`. |
| fail2ban         | Ban enforcement                      | Works alongside CrowdSec (verify active policy) |

Verify current CrowdSec/fail2ban state with: `sudo systemctl status crowdsec`, `sudo fail2ban-client status all` on VPS.

