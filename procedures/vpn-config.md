# VPN Configuration — WireGuard Primary (Synced 2026-11-16)

### Overview

1. **WireGuard** is the primary overlay network for private communication across hosts and services.
   - Server: Cloud VPS (`wg0`, `10.9.0.1/24`)
   - Peers: Raspberry Pi 5, Orange Pi, H81 (if enabled), Garuda desktop, Spectre x360, mobile devices

2. **Tailscale** (historical) — Fallback mesh overlay for CGNAT-traversing admin access only when needed. Not confirmed active in current state.

3. **Gluetun**— Commercial VPN container for torrents on Pi to isolate torrent IP from home network.

---

## WireGuard Server Config (VPS `/etc/wireguard/wg0.conf`)

> ⚠️ Below are structure examples only — never store real keys here. Rotate all keys periodically and verify connectivity after changes.

```
[Interface]
Address = 10.9.0.1/24
ListenPort = 51820
PrivateKey = [PLACEHOLDER: VPS_PRIVATE_KEY]

# Routing to local home LAN (may not be needed depending on network setup)
PostUp = iptables -A FORWARD -i %i -j ACCEPT; PostUp = iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
PreDown = iptables -D FORWARD -i %i -j ACCEPT; PreDown = iptables -t nat -D POSTROUTING -o eth0 -j MASQUERADE

# Peer entries (one per device)
[Peer]
PublicKey = [PEER_PUBLIC_KEY]           # Generated from client: wg pubkey < private_key_file
PresharedKey = [OPTIONAL_PRESHARED_KEY] # Optional for added security
AllowedIPs = 10.9.0.2/32                # Example Peer IP assignment (verify against ip-assignments.md)

# Repeat for each peer...
```

---

## WireGuard Client Config Examples (peer devices)

Each client (`wg0.conf`) should point to VPS `Endpoint` with its public key:

### Raspberry Pi 5

```
[Interface]
PrivateKey = [PLACEHOLDER: PI_WG_PrivateKey]
Address = 10.9.0.4/24
#DNS = 10.9.0.1                        # Optional: DNS settings per device requirements

[Peer]
PublicKey = [VPS_PUBLIC_KEY]
PresharedKey = [OPTIONAL_PRE_SHARED_KEY]
Endpoint = [PLACEHOLDER_VPS_PUBLIC_IP]:51820   # For remote access across ISP NAT/CGNAT
AllowedIPs = 10.9.0.1/24

PersistentKeepalive = 25               # Keep tunnel alive to allow endpoint discovery
---

## Enabling and Managing WireGuard

### VPS — Setup Commands

```bash
# Install (Debian / Ubuntu)
sudo apt install wireguard

# Generate server side key pair
wg genkey | sudo tee /etc/wireguard/server_private.key
wg pub  <  /etc/wireguard/server_private.key | sudo tee /etc/wireguard/server_public.key
chmod  600 /etc/wireguard/*.key

# Start and enable on boot
sudo wg-quick up wg0
sudo systemctl enable --now wg-quick@wg0

# Show status and connected peers
sudo wg show
```

### Clients

Same key generation commands; then `sudo wg-quick up wg0` along with `sudo wg` to test — verify handshakes.

---

## Gluetun VPN (qBittorrent)

- Isolates torrent traffic through a commercial VPN (Surfshark) to bypass ISP throttling and prevent IP leaks
- Container `gluetun` requires kernel capability: `NET_ADMIN`
- qBittorrent uses `network_mode: "service:gluetun"` to inherit the VPN interface

### Verification Commands

```bash
# Check Gluetun container is healthy
docker inspect gluetun --format '{{.State.Health.Status}}'

# Verify torrent client is bound to VPN
docker exec qbittorrent curl https://api.ipify.org
# This should show the VPN exit node IP, not the Pi's public IP
```

---

## Troubleshooting (Common Issues)

### WireGuard Handshake Failures

1. Confirm keys match: server's `[Peer] PublicKey` must exactly match client's public key
2. Verify `AllowedIPs` includes peer IP ranges on both sides
3. Check endpoint reachability: `ping -c 3 [VPS_PUBLIC_IP]`
4. Confirm VPS allow forwarding: `sysctl net.ipv4.conf.all.forwarding=1`
5. Look for firewall rules on the VPS blocking `51820/udp`:
   `sudo ufw status | grep 51820`

### Keepalive Drops

- PersistentKeepalive = 25 recommended for mobile/NAT clients, 120 for always-on LAN devices.
- Check `sudo wg` last handshake — should be < 2 minutes for connected peers.

### Tailscale as Backup

- Only for out-of-band administration. Keep disabled by default.
- Install: `curl -fsSL https://tailscale.com/install.sh | sh`
- Auth is via browser or auth key.

Refer to network/ip-assignments.md for peer inventory, current AllowedIPs and DNS server configs.
