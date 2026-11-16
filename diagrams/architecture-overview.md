# Architecture Diagram Reference — Narrative Walkthrough

> This document narrates the homelab network topology as visualized in [`Updated Homelab Architecture v3.drawio.png`](./Updated%20Homelab%20Architecture%20v3.drawio.png).  
> Values marked "historical" require verification.

---

## North-South Traffic (External → Internal)

```
User requests external service (nextcloud.example.com → VPS public IP)
  │
  ├── Cloudflare DNS resolves A-record to VPS IP
        │
        ▼
      [Caddy on VPS]
      - TLS termination at :443/TCP
      - Matches virtual host, routes via WireGuard tunnel:
          reverse_proxy http://10.9.0.4:8080           # Nextcloud example
        │
        └──[WireGuard tunnel (wg0, 10.9.0.x)]─────────┐
                                                        ▼
                                                   Raspberry Pi 5
                                                   Service: nextcloud container
```

**VPS-Hosted Services** (no tunnel needed — on same host as Caddy):

- Dashy (`home`), Vaultwarden (`vault.`), Uptime Kuma, Portainer (vps instance)

---

## Pi-Hosted Services (via WG Tunnel Accessible from Any Peer)

### Example: External Access to Immich

```
User → Cloudflare DNS for immich.example.com → VPS IP:443
      ↓ Caddy matches domain → Routes to WG peer 10.9.0.4:2283 (Pi)
      └── Immich server receives HTTPS request
          ↓ Internal container network (media_net)
Immich microservices, database, ML containers communicate via docker overlay
```

---

## Local LAN Resolution (Pi-hole on VPS)

Devices and peers configure DNS to point to the VPS WireGuard IP `10.9.0.1`

**Example:**

```
HP Spectre x360 (WireGuard peer, not homelab node)
  └── DNS query: nextcloud.example.com → Sent to 10.9.0.1
       │
       ├── VPS receives query over wg0
            ├── Forwards to Pi-hole container on VPS
                  ├── Resolves and applies blocklists or private rewrites
                  └── Returns IP (VPS public IP for nextcloud)
```

---

## East-West Internal Routing

```
[Pi 5 media_net container]────→ [Immich DB]    ← Internal Docker overlay
                                ↔ [Nextcloud]   ← Same bridge network allows direct comms
```

---

## qBittorrent + Gluetun VPN Isolation

```
qBittorrent container → shares namespace with Gluetun
                        │
                        ├── Surfaces at :8080 (Web UI) reachable from Peers via WG
                        └── Torrent peer traffic → routed through Surfshark tunnel
                             Exit node: not the home network
```

---

## AI Data Flow — Bifrost Gateway Model

```
OpenWebUI (Garuda desktop, 10.9.0.2)
  │
  ↓ HTTP (endpoint ai.aryadivap.com on VPS Caddy)
Bifrost gateway (Orange Pi, v2.3.2)
  │
  ├── Routing decision: local vs hosted provider:
        - OpenRouter API (z-ai/glm-5-flash), or
        - Local llama.cpp ROCm container (Garuda desktop peer)
  │
  └── Response streamed back to user client
```

**MCP Tools:** Bifrost provides MCP server endpoints; OpenWebUI connects via `https://bifrost.aryadivap.com/mcp/core-mcp` per historical configuration.

