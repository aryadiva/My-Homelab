# Power Outage Recovery

## Expected Boot Order

After a power outage, the system must be restored in the correct sequence to avoid data corruption or service startup failures:

```
1. Network Infrastructure (ER605 + switches)
2. Raspberry Pi 5
3. External Storage (LVM mount)
4. Docker Engine
5. Containerized Services
6. Cloud VPS (WireGuard tunnel reconnect)
7. Garuda Desktop (if also powered off)
```

---

## Step-by-Step Recovery

### 1. Network Infrastructure

- **ER605:** Should boot automatically when power is restored. Verify LEDs for WAN1, WAN2, and LAN activity.
- **Managed Switch:** Should boot with the router. Verify LAN connectivity.
- Wait ~2 minutes for the router to establish WAN connections (dual-ISP negotiation).

### 2. Raspberry Pi 5

- The Pi should be configured to boot automatically on power restore (check `config.txt` or `eeprom` settings).
- **If not booting:** Check the USB-C power supply and SD card / SSD connection.
- **Connect via SSH:**
  ```bash
  ssh user@10.0.0.[PLACEHOLDER]
  ```
- **Verify system health:**
  ```bash
  uptime
  dmesg | tail -20
  df -h
  free -h
  ```

### 3. External Storage — LVM Mount

The LVM volume must be mounted before Docker services start.

```bash
# Check if volume group is available
sudo vgscan
sudo vgchange -ay

# Check if logical volume is active
sudo lvscan

# Mount if not mounted
ls /mnt/lvm  # Should show contents
# If empty, mount manually:
sudo mount /dev/[PLACEHOLDER: vg name]/[PLACEHOLDER: lv name] /mnt/lvm

# Verify FUSE mount if applicable
# [PLACEHOLDER: add any FUSE-specific commands if the docking enclosure uses mergerfs or similar]
```

**If LVM fails to activate:**
```bash
# Check physical volumes
sudo pvscan
# Check volume groups
sudo vgscan --mknodes
# Force activation if safe
sudo vgchange -ay --force
```

### 4. Docker Engine

```bash
# Verify Docker is running
sudo systemctl status docker
# If not running:
sudo systemctl start docker

# Check all containers are stopped (they should be; Docker restarts policy will handle them)
docker ps -a
```

### 5. Container Startup

Docker containers with `restart: unless-stopped` or `restart: always` will start automatically with the Docker daemon. However, startup order matters:

```bash
# Check all containers started successfully
docker ps

# If some are missing, start them manually:
docker compose up -d

# Verify each service:
# - Homepage: http://10.0.0.x:3000
# - Portainer: http://10.0.0.x:9000
# - Pi-hole: http://10.0.1.x/admin (VPS — accessible over WireGuard)
# - Uptime Kuma: http://10.0.0.x:3001
# - Nextcloud: http://10.0.0.x:8080
# - Immich: http://10.0.0.x:2283
# - Jellyfin: http://10.0.0.x:8096
```

**Known startup dependency order:**
```
Gluetun (VPN) → qBittorrent (needs Gluetun healthy)
Nextcloud DB + Redis → Nextcloud (needs DB ready)
Immich DB + Redis → Immich server + microservices + ML
```

### 6. Cloud VPS

- The VPS should boot and reconnect automatically.
- **Verify WireGuard tunnel:**
  ```bash
  # On Pi
  sudo wg show
  # On VPS
  sudo wg show
  ```
- **Expected:** Handshake count should increase; latest handshake should be recent.

### 7. Garuda Desktop (if applicable)

- Boot normally.
- Verify Ollama and Open WebUI are running:
  ```bash
  systemctl --user status ollama
  systemctl --user status open-webui
  ```

---

## Quick Health Check Script

```bash
#!/bin/bash
# health-check.sh
echo "=== System Health ==="
echo "Uptime: $(uptime -p)"
echo "CPU Temp: $(vcgencmd measure_temp)"
echo "Disk: $(df -h /mnt/lvm | tail -1)"
echo "Docker: $(docker ps --format '{{.Names}}' | wc -l) containers running"
echo "WireGuard: $(sudo wg show | grep -c handshake) active peers"
# Pi-hole is on VPS — run from VPS or via SSH:
# echo "Pi-hole: $(ssh vps 'docker exec pihole pihole status' | grep -c enabled)"
```

---

## If Power Loss Was Unexpected

- **Check file system integrity** on the LVM volume:
  ```bash
  sudo umount /mnt/lvm
  sudo fsck /dev/[PLACEHOLDER: vg/lv]
  sudo mount /dev/[PLACEHOLDER: vg/lv] /mnt/lvm
  ```

- **Check PostgreSQL integrity** (Nextcloud + Immich):
  ```bash
  docker exec nextcloud-db psql -U nextcloud -c "SELECT now();"
  docker exec immich-db psql -U immich -c "SELECT now();"
  ```

- **Check Redis AOF/RDB** for corruption:
  ```bash
  docker logs nextcloud-redis | tail -10
  docker logs immich-redis | tail -10
  ```

---

## Prevention

- [ ] Configure Pi to boot on power restore (already done? [PLACEHOLDER])
- [ ] Add `/etc/fstab` entry for LVM auto-mount
- [ ] Consider UPS for graceful shutdown:
  ```bash
  # Example: configure NUT client for USB UPS
  sudo apt install nut-client
  # Configure /etc/nut/upsmon.conf for auto-shutdown at low battery
  ```
