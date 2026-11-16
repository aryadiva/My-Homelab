# IP Assignments — Synced 2026-11-16

> Historical peer values retained from earlier docs; KB marks them as unverified.
> Confirm peers and IPs with live `wg show` before relying on network configs.

---

## Physical LAN Subnet — `192.168.10.0/24`

| Device | Interface | IP Address | Notes |
|--------|-----------|------------|-------|
| ER605 Router | LAN | `192.168.10.1` | Default gateway, DHCP for physical LAN |
| Raspberry Pi 5 | eth0 | `192.168.10.188` | Static reservation |
| Garuda Desktop | eth0 | `192.168.10.201` | Static reservation |

> Physical LAN managed by consumer router; listed for documentation continuity. Verify with `ip addr` on each device.

---

## WireGuard Overlay — `10.9.0.0/24`

**VPS (server)**: `10.9.0.1/24`, listen `51820/udp`

### Peer Inventory (historical, unverified)

| Client | WireGuard IP | Endpoint | AllowedIPs | Notes |
|--------|--------------|----------|------------|-------|
| Raspberry Pi 5 | `10.9.0.4` | VPS public IP | `10.9.0.0/24` | Primary on-prem node |
| Garuda Desktop | `10.9.0.2` | VPS public IP | `10.9.0.0/24, 192.168.10.0/24` | GPU / AI — llama.cpp endpoint |
| Windows Laptop | `10.9.0.3` | VPS public IP | `10.9.0.0/24` | Remote client |
| Samsung Phone | `10.9.0.5` | VPS public IP | `10.9.0.0/24` | Mobile peer |
| Orange Pi 5 Pro | `10.9.0.6` | VPS public IP | `10.9.0.0/24` | Bifrost gateway |

> ⚠️ KB does not confirm exact peer IPs / AllowedIPs. Values above are historical. Verify with `sudo wg show` on the VPS.

**Tailscale**: documented historically (`arya-ubuntu`, `arya-garuda`) but **not confirmed deployed**. Reference only.

---

## DNS

- **Resolver**: Pi-hole on VPS, reached over WireGuard at `10.9.0.1`
- **Upstream**: Cloudflare `1.1.1.1` and Quad9 `9.9.9.9`
- **Local domain**: `aryadivap.com`

All peers use `10.9.0.1` as DNS. Verify with `dig @10.9.0.1 aryadivap.com`.

---

## Reverse Proxy Mappings (historical)

| Subdomain | Target | Notes |
|-----------|--------|-------|
| `bifrost.aryadivap.com` | Orange Pi — Bifrost gateway | Domain confirmed in KB |
| `nextcloud.aryadivap.com` | `http://10.9.0.4:8080` | MariaDB + Redis backend |
| `ai.aryadivap.com` | `http://10.9.0.2:3000` | OpenWebUI on Garuda |
| `portainer-home.aryadivap.com` | Pi — Portainer Server | Main Docker control plane |
| `immich.aryadivap.com` | `http://10.9.0.4:2283` | Immich app |
| `jellyfin.aryadivap.com` | `http://10.9.0.4:8096` | Jellyfin media server |
| `matrix.aryadivap.com` | `http://10.9.0.4:8008` | Synapse client-server API |
| `element.aryadivap.com` | `http://10.9.0.4:8008` | Element web via subpath |
| `pihole.aryadivap.com` | VPS local (`127.0.0.1:80`) | Pi-hole admin |
| `home.aryadivap.com` | VPS local (`10.9.0.1:8080`) | Dashy |
| `vault.aryadivap.com` | VPS local (`10.9.0.1:8989`) | Vaultwarden |
| `uptime.aryadivap.com` | VPS local (`10.9.0.1:3001`) | Uptime Kuma |

### Path corrections (KB)

- Nextcloud data path: **`/mnt/lvm/nextcloud-data`** (corrected from old `/mnt/lvm/nextcloud/data` doc).
- Immich intended path: **`/mnt/lvm/immich-app`** (verify live — historical showed `/mnt/lvm/immich/photos`).

---

## Known Port Conflicts (verify before changes)

- Historical config shows Nextcloud and qBittorrent Web UI both on `10.9.0.4:8080`.

Before editing `.env`, port bindings, or firewall rules, verify with live diagnostics: `docker ps`, `ss -ltn | grep LISTEN`, `ufw status`, `wg show wg0`)
