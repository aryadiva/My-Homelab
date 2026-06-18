# Deployment Procedure

## Standard Service Update Workflow

Used when updating an existing service (new image tag, config change, or env var update).

### Step 1 — Pull Latest Images

```bash
cd /path/to/docker-compose-directory
docker compose pull <service-name>
```

Or pull all:

```bash
docker compose pull
```

### Step 2 — Recreate the Container

```bash
docker compose up -d <service-name>
```

Or recreate all:

```bash
docker compose up -d
```

Docker Compose will only recreate services whose image or configuration has changed.

### Step 3 — Verify Health

```bash
# Check container status
docker ps --filter "name=<service-name>"

# Check logs for errors
docker compose logs <service-name> --tail=50

# Check the service URL/port
curl -I http://localhost:<port>
```

### Step 4 — Clean Up Old Images (Optional)

```bash
docker image prune -a
```

---

## Rolling Back a Service

If a new version causes issues:

```bash
# Tag the previous image version in docker-compose.yml
# or use a specific tag:
docker compose up -d <service-name>

# If using latest, pull the previous tag explicitly:
# 1. Edit docker-compose.yml to pin the previous version tag
# 2. docker compose up -d <service-name>
```

**Quick rollback reference:**

| Service | Previous Stable Tag |
|---------|-------------------|
| Nextcloud | `v32` |
| Immich | `v2.7.4` |
| Jellyfin | `v10.11.8` |
| Matrix Synapse | `latest` |
| Pi-hole | `v6.4` |

---

## Adding a New Service

1. **Add the service block** to the appropriate `docker-compose.yml` section (infra or media).
2. **Assign it to the correct Docker network.**
3. **Choose a port** that does not conflict (check `services/inventory.md` for port allocations).
4. **Update** `services/inventory.md` and `services/container-network-map.md` with the new entry.
5. **Update Pi-hole** with a local DNS record if needed.
6. **Deploy:**
   ```bash
   docker compose up -d <new-service>
   ```

---

## Environment Variable Management

Sensitive values (DB passwords, API keys, WireGuard keys) should be stored in:

1. A `.env` file in the Compose directory (excluded from Git via `.gitignore`)
2. Referenced in `docker-compose.yml` as `${VARIABLE_NAME}`

```bash
# .env file template (DO NOT COMMIT)
NEXTCLOUD_DB_PASSWORD=your_password_here
IMMICH_DB_PASSWORD=your_password_here
PIHOLE_WEBPASSWORD=your_password_here
WIREGUARD_PRIVATE_KEY=your_key_here
TZ=Asia/Jakarta
```

---

## Maintenance Mode (Nextcloud)

```bash
# Enable
docker exec nextcloud occ maintenance:mode --on

# Disable
docker exec nextcloud occ maintenance:mode --off
```

---

## Service-Specific Notes

### qBittorrent + Gluetun

- qBittorrent **must** start after Gluetun is healthy.
- To restart Gluetun (e.g., switch VPN region):
  ```bash
  docker compose restart gluetun
  # qBittorrent will restart automatically due to restart policy
  ```

### Jellyfin

- Media is mounted **read-only** (`:ro` suffix in volume mount).
- To add new media, write to `/mnt/lvm/media/` directly (not through Jellyfin) through nextcloud or save files from qBittorrent.

### Pi-hole

- Restarting Pi-hole will briefly interrupt DNS resolution.
- Plan updates during low-traffic windows.
