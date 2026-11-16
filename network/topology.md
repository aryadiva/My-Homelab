# Network Topology Summary — Synced 2026-11-16

> Historical documentation retained for reference; verify against live `wg show`, UFW status, and service listening before high-impact changes.

---

## Core Architecture Constraints

**CGNAT home edge**: Home ISP provides CGNAT-only egress — no inbound public ports on the home network. All external ingress (reverse proxy, direct services) routes through the Cloud VPS edge node.

**WireGuard overlay**: Primary private networking via WireGuard tunnel. The VPS runs the server; all other machines connect as peers.

### Physical vs Logical Separation

- **Physical LAN** — managed by consumer router, DHCP, local subnet: `192.168.10.0/24` (Ovall/Bnet dual-ISP load balancing reported historically).
- **WireGuard overlay** — encrypted private network `10.9.0.0/24`, peers communicate through VPS server.

> ⚠️ The ER605 router exists but is **not in the KB's verified scope**. Keep for historical continuity; verify routing/policies before changing it.

---

## WireGuard Overlay — Live Topology (Verify Before Changes)

| Host               | Role                          | WG Interface     | Subnet        | Notes                                |
|--------------------|-------------------------------|------------------|---------------|--------------------------------------|
| Cloud VPS          | Server, ingress edge          | wg0              | 10.9.0.1/24   | Confirmed; listen port 51820         |
| Raspberry Pi 5     | Peer (Pi5 primary node)       | wg0              | ?             | `AllowedIPs` include Pi services     |
| Orange Pi 5 Pro    | Peer (Bifrost gateway)        | wg0              | ?             | See historical docs for exact IP     |
| H81H3-I x86 Mini PC| Peer (role unsettled)         | wg0              | ?             | Verify if enabled over WireGuard     |
| AMD Desktop        | Peer (local AI inference)     | wg0              | ?             | llama.cpp reachable via Bifrost      |
| HP Spectre x360    | Personal laptop peer          | wg0              | ?             | Access Pi-hole only; unverified IP   |

> ⚠️ **Peer IPs are historical** — KB does NOT confirm exact addresses. Use `wg show` on VPS to verify current peers, AllowedIPs and status.

**Listen port**: `51820/udp` (VPS UFW rule allows this from any).

Tailscale is documented but **not confirmed deployed** in KB scope — keep for fallback access reference only.

---

## Service Placement via WireGuard

Services hosted on the Raspberry Pi are reachable over WireGuard from anywhere that can reach the VPS:

| Service Host        | Sample Routes Through WG Tunnel To                                  |
|---------------------|----------------------------------------------------------------------|
| Raspberry Pi 5      | Caddy reverse proxy forwards `nextcloud`, `jellyfin`, `immich` etc. |
| Cloud VPS           | Dashy, Vaultwarden, Uptime Kuma (same host — no tunnel needed)     |
| AMD Desktop         | llama.cpp endpoint → Bifrost routes to OpenRouter or local          |

---

## Firewall & Port Summary

### Cloud VPS — UFW Policy

**Default**: deny incoming / allow outgoing  
**Allow from 10.9.0.0/24 (WireGuard peers)**:

| Port    | Protocol | Source               | Notes                              |
|---------|----------|----------------------|------------------------------------|
| 51820   | UDP      | `0.0.0.0/0`          | WireGuard listen port              |
| 443     | TCP/UDP  | `0.0.0.0/0`          | HTTPS + HTTP/3 (QUIC) for Caddy    |
| 80      | TCP      | `0.0.0.0/0`          | HTTP → redirect to `443`           |
| 22      | TCP      | `0.0.0.0/0` LIMIT    | SSH rate-limited for safety        |

Docker service ports on VPS (Caddy, Dashy, Vaultwarden) bind to **WireGuard IP (`10.9.0.1`)** instead of all interfaces — per security guidance in KB.

### Peer Traffic Rules

- **Peer to Pi-hosted services**: Port binding uses `host_ip:port` via tunnel (e.g., `http://10.9.0.4:8080`).
- **DNS (Pi-hole)**: VPS hosts Pi-hole; peers resolve to `10.9.0.1`.

---

## External Routing Diagram (Logical)

```
Internet → VPS public edge
        │
        ├── 443/80/TCP+UDP   → Caddy reverse proxy (Pi services via WG)
        ├── 51820/UDP        → WireGuard server
        └── Private peers    → Tunnel ↔ each other over 10.9.0.0/24
```

DNS upstreams from Pi-hole: `1.1.1.1` and `9.9.9.9`.

---

## Security Services on VPS

| Service         | Function                            | Notes                                     |
|-----------------|-------------------------------------|-------------------------------------------|
| Caddy           | Reverse proxy / TLS edge            | Auto HTTPS; routes to Pi over WireGuard   |
| CrowdSec        | Log acquisition + bouncer           | Previously had connection refused errors  |
| fail2ban        | Ban policy enforcement              | Active alongside CrowdSec (verify config) |
| Uptime Kuma     | Service health monitoring           | HTTP/TCP/ping checks enabled              |
| Dashy           | Dashboard portal                    | Replaces deprecated Homepage              |
| Vaultwarden     | Password vault (Bitwarden clone)    |                       

---

## Traffic Patterns

### External request to Pi service

> User → Cloudflare DNS A-record (`nextcloud.aryadivap.com` → VPS public IP)  
> → VPS Caddy listens 443, matches domain, reverses proxy over WireGuard tunnel  
> → `http://10.9.0.4:8080` (Nextcloud)

### Local resolution via Pi-hole

> LAN device / Peer DNS set to VPS WG IP `10.9.0.1`  
> → DNS query through WG tunnel → VPS Pi-hole container  
> → Upstream: Cloudflare (`1.1.1.1`) and Quad9 (`9.9.9.9`)

### qBittorrent with Gluetun VPN isolation

> qBittorrent binds to `gluetun` service namespace  
> → Uses same Surfshark WireGuard tunnel (not VPS WG)  
> → Tunnel egress routes download traffic, preventing IP leakage

---

## Known Unverified Values — Verify Before Production Changes

The KB marks these items as "uncertain" or absent from the snapshot:

- Current active peer list / AllowedIPs
- Whether Tailscale is actually enabled and which peers are in mesh
- Port binding for Pi-hosted services (historical docs show `10.9.0.x:PORT`)
- Whether all historical Compose services remain deployed (`Forgejo`, `Time-MCP`, `Authentik`, etc.)

Before applying network changes, re-verify the live state on VPS and each host.
