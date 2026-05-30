# VPN Configuration

## Overview

Three VPN layers exist in this infrastructure:

1. **WireGuard** — VPS is the WireGuard server; all LAN devices (Pi, Garuda, laptops) are clients connecting through it for secure overlay networking and service access.
2. **Tailscale** — Fallback CGNAT-traversing mesh for out-of-band admin access.
3. **Gluetun (Surfshark)** — Commercial VPN for torrent traffic isolation on the Pi.

---

## 1. WireGuard (VPS Server / LAN Clients)

### Architecture

```
Cloud VPS (public IP) ─── WireGuard Server (10.0.1.1)
      │
      ├── Raspberry Pi 5 (client)      ─── 10.0.1.x
      ├── Garuda Desktop (client)      ─── 10.0.1.x
      ├── Laptop / Phone (client)      ─── 10.0.1.x
      └── ... other LAN devices
```

The VPS runs the WireGuard server natively (not in Docker). All other devices run WireGuard as clients and connect to the VPS. The VPS routes traffic between peers and provides access to Pi-hole DNS and Caddy-proxied services.

### VPS Configuration (`/etc/wireguard/wg0.conf`)

```
[Interface]
PrivateKey = [PLACEHOLDER: VPS private key]
Address = 10.0.1.1/24
ListenPort = [PLACEHOLDER: e.g., 51820]

# Enable IP forwarding
net.ipv4.ip_forward = 1

# Raspberry Pi 5
[Peer]
PublicKey = [PLACEHOLDER: Pi public key]
AllowedIPs = 10.0.1.2/32, 10.0.0.0/24

# Garuda Arch Desktop
[Peer]
PublicKey = [PLACEHOLDER: Garuda public key]
AllowedIPs = 10.0.1.3/32

# [PLACEHOLDER: additional peers]
```

### Client Configuration (Raspberry Pi 5 — `/etc/wireguard/wg0.conf`)

```
[Interface]
PrivateKey = [PLACEHOLDER: Pi private key]
Address = 10.0.1.2/32
DNS = 10.0.1.1  # Pi-hole on VPS

[Peer]
PublicKey = [PLACEHOLDER: VPS public key]
Endpoint = [PLACEHOLDER: VPS public IP]:[PLACEHOLDER: port]
AllowedIPs = 10.0.1.0/24
PersistentKeepalive = 25
```

### Client Configuration (Garuda Desktop — `/etc/wireguard/wg0.conf`)

```
[Interface]
PrivateKey = [PLACEHOLDER: Garuda private key]
Address = 10.0.1.3/32
DNS = 10.0.1.1  # Pi-hole on VPS

[Peer]
PublicKey = [PLACEHOLDER: VPS public key]
Endpoint = [PLACEHOLDER: VPS public IP]:[PLACEHOLDER: port]
AllowedIPs = 10.0.1.0/24
PersistentKeepalive = 25
```

### Client Configuration (Laptop / Mobile)

Use the official WireGuard apps. Create a config file with:

```
[Interface]
PrivateKey = [PLACEHOLDER: client private key]
Address = 10.0.1.x/32
DNS = 10.0.1.1  # Pi-hole on VPS

[Peer]
PublicKey = [PLACEHOLDER: VPS public key]
Endpoint = [PLACEHOLDER: VPS public IP]:[PLACEHOLDER: port]
AllowedIPs = 10.0.1.0/24
PersistentKeepalive = 25
```

> **Note:** If you want to route all device traffic through the VPS (full-tunnel), change `AllowedIPs` to `0.0.0.0/0, ::/0`. The current config uses split-tunnel (only WireGuard subnet traffic goes through the tunnel).

### Key Generation

```bash
# Generate key pair on each device
wg genkey | tee privatekey | wg pubkey > publickey
```

### Service Management

```bash
# Start WireGuard
sudo wg-quick up wg0

# Stop WireGuard
sudo wg-quick down wg0

# Show status
sudo wg show

# Enable on boot
sudo systemctl enable wg-quick@wg0

# On VPS: verify peers are connected
sudo wg show wg0
# Look for "latest handshake" timestamps for each peer
```

---

## 2. Tailscale (Fallback Mesh)

### Setup

```bash
# Install on each node:
curl -fsSL https://tailscale.com/install.sh | sh

# Authenticate:
sudo tailscale up

# On headless devices (Pi, VPS):
sudo tailscale up --authkey=[PLACEHOLDER: auth key from Tailscale admin console]
```

### Configuration

- **Subnet routing** (if desired):
  ```bash
  sudo tailscale up --advertise-routes=10.0.0.0/24 --accept-routes
  ```

- **ACLs** are managed via [Tailscale Admin Console](https://login.tailscale.com).

### Key Features Used

- **CGNAT traversal:** Works when WireGuard is blocked or unreachable
- **Out-of-band admin:** SSH into Pi via Tailscale IP if LAN is down
- **Secondary access path** for services

---

## 3. Gluetun (qBittorrent VPN)

### Docker Compose (see also `services/docker-compose-overview.md`)

```yaml
gluetun:
  image: qmcgaw/gluetun:latest
  container_name: gluetun
  ports:
    - "[PLACEHOLDER_HOST_IP]:8080:8080"
  volumes:
    - ./gluetun/config:/gluetun
  environment:
    VPN_SERVICE_PROVIDER: surfshark
    VPN_TYPE: wireguard
    WIREGUARD_PRIVATE_KEY: "[PLACEHOLDER]"
    WIREGUARD_ADDRESSES: "[PLACEHOLDER]"
    SERVER_REGIONS: "[PLACEHOLDER: e.g., Australia]"
  cap_add:
    - NET_ADMIN
  restart: unless-stopped

qbittorrent:
  image: lscr.io/linuxserver/qbittorrent:latest
  container_name: qbittorrent
  network_mode: "service:gluetun"
  volumes:
    - ./qbittorrent/config:/config
    - /mnt/lvm/downloads:/downloads
  environment:
    PUID: "[PLACEHOLDER]"
    PGID: "[PLACEHOLDER]"
    TZ: "[PLACEHOLDER: timezone]"
    UMASK: "002"
  depends_on:
    gluetun:
      condition: service_healthy
  restart: unless-stopped
```

### Switching VPN Region

```bash
# Update the SERVER_REGIONS env var in docker-compose.yml
# Then recreate:
docker compose up -d gluetun
```

### Verifying VPN Connection

```bash
# Check Gluetun status
docker logs gluetun --tail 20

# Verify external IP from within qBittorrent's namespace
docker exec qbittorrent curl -s ifconfig.me

# Expected: Surfshark exit node IP, NOT your home IP
```

### Kill Switch

Gluetun has a built-in kill switch. If the VPN drops:

- All traffic through the Gluetun network namespace is blocked
- qBittorrent loses connectivity to prevent IP leakage
- Gluetun automatically reconnects and resumes traffic

---

## Troubleshooting

### WireGuard Handshake Failing

```bash
# On client (e.g., Pi):
ping 10.0.1.1  # VPS tunnel IP
sudo wg show
sudo tcpdump -i wg0 -n

# On VPS:
sudo wg show
# Check all peers have recent handshakes
sudo ufw status  # Check firewall allows WireGuard port
```

### Client Cannot Reach Pi Services

```bash
# Verify client is connected to WireGuard
sudo wg show

# Verify client DNS is set to 10.0.1.1
# From client:
nslookup nextcloud.example.com 10.0.1.1

# Verify VPS can reach the Pi over WireGuard
# From VPS:
ping 10.0.1.x  # Pi WireGuard IP
curl http://10.0.1.x:8080  # Test Nextcloud directly
```

### Gluetun Connection Issues

```bash
# View detailed logs
docker compose logs gluetun

# Test DNS from within the Gluetun namespace
docker exec gluetun ping 1.1.1.1

# Recreate with fresh config
docker compose down gluetun
docker compose up -d gluetun
```

---

> **Note:** All private keys and public IPs have been replaced with placeholders. The actual files on disk contain live values and should never be committed to Git.
