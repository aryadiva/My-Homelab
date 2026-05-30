# Backup & Restore Procedures

## Overview

This document covers backup and restore workflows for all containerized services. Backups fall into two categories:

1. **Database dumps** — PostgreSQL and any other DB engines
2. **Volume backups** — Application data, config, and media files

---

## Backup Schedule

| What | Frequency | Method | Retention |
|------|-----------|--------|-----------|
| Nextcloud DB + data | Daily | `pg_dump` + rsync | 7 days local |
| Immich DB + photos | Daily | `pg_dump` + rsync | 7 days local |
| Docker Compose files | Per-change | Git commit | Full Git history |
| Media files (Jellyfin) | Weekly | rsync | 2 copies |
| Config dirs (`./prowlarr`, `./radarr`, etc.) | Weekly | tar | 30 days |
| Full system state | Monthly | `docker-compose config` + volume snapshots | 3 months |

[PLACEHOLDER: add off-site backup destination if applicable, e.g., Backblaze B2, rsync.net, or second external drive]

---

## Database Dumps

### PostgreSQL (Nextcloud, Immich)

```bash
#!/bin/bash
# backup-postgres.sh
BACKUP_DIR="/mnt/lvm/backups/postgres"
DATE=$(date +%Y%m%d_%H%M%S)

# Nextcloud
docker exec nextcloud-db pg_dump -U nextcloud nextcloud > "$BACKUP_DIR/nextcloud_$DATE.sql"

# Immich
docker exec immich-db pg_dump -U immich immich > "$BACKUP_DIR/immich_$DATE.sql"

# Compress
gzip "$BACKUP_DIR/nextcloud_$DATE.sql"
gzip "$BACKUP_DIR/immich_$DATE.sql"

# Remove backups older than 7 days
find "$BACKUP_DIR" -name "*.sql.gz" -mtime +7 -delete
```

### Redis (Cache Only)
- Redis is ephemeral cache; no backup needed.
- On restart, Nextcloud and Immich will repopulate the cache.

---

## Volume Backup

### Application Data Volumes

```bash
#!/bin/bash
# backup-volumes.sh
BACKUP_DIR="/mnt/lvm/backups/volumes"
DATE=$(date +%Y%m%d)

tar czf "$BACKUP_DIR/nextcloud_data_$DATE.tar.gz" -C /mnt/lvm/nextcloud data
tar czf "$BACKUP_DIR/immich_photos_$DATE.tar.gz" -C /mnt/lvm/immich photos

# Config directories
# Note: Pi-hole config is on the VPS — backup separately from the VPS host
# SSH into VPS and run: tar czf pihole_config.tar.gz ./pihole
tar czf "$BACKUP_DIR/configs_$DATE.tar.gz" \
  ./prowlarr \
  ./radarr \
  ./qbittorrent \
  ./jellyfin
```

---

## Restore Procedures

### Nextcloud Full Restore

1. **Stop the containers:**
   ```bash
   docker compose stop nextcloud nextcloud-db nextcloud-redis
   ```

2. **Restore database:**
   ```bash
   # Start DB only
   docker compose start nextcloud-db
   sleep 5
   docker exec -i nextcloud-db psql -U nextcloud nextcloud < nextcloud_20250101_120000.sql
   ```

3. **Restore data files:**
   ```bash
   rm -rf /mnt/lvm/nextcloud/data/*
   tar xzf /mnt/lvm/backups/volumes/nextcloud_data_20250101.tar.gz -C /mnt/lvm/nextcloud
   ```

4. **Restore config and apps:**
   ```bash
   # Config is in ./nextcloud/config
   # Apps are in ./nextcloud/custom_apps
   # Restore from your backup if needed
   ```

5. **Restart and verify:**
   ```bash
   docker compose up -d nextcloud nextcloud-redis
   docker exec nextcloud occ maintenance:mode --off
   docker exec nextcloud occ files:scan --all
   ```

### Immich Full Restore

1. **Stop the containers:**
   ```bash
   docker compose stop immich-server immich-microservices immich-machine-learning immich-db immich-redis
   ```

2. **Restore database:**
   ```bash
   docker compose start immich-db
   sleep 5
   docker exec -i immich-db psql -U immich immich < immich_20250101_120000.sql
   ```

3. **Restore photos:**
   ```bash
   rm -rf /mnt/lvm/immich/photos/*
   tar xzf /mnt/lvm/backups/volumes/immich_photos_20250101.tar.gz -C /mnt/lvm/immich
   ```

4. **Restart:**
   ```bash
   docker compose up -d immich-server immich-microservices immich-machine-learning immich-redis
   ```

### Individual Service Restore (Stateless)

For services like Prowlarr, Radarr, Jellyfin, Pi-hole, Uptime Kuma, and Homepage:

```bash
# 1. Stop the container
docker compose stop <service-name>

# 2. Restore config from backup
tar xzf /mnt/lvm/backups/volumes/configs_$DATE.tar.gz ./<service-name>

# 3. Restart
docker compose up -d <service-name>
```

---

## Backup Verification

- **Monthly**: Test-restore Nextcloud + Immich to a staging directory
- **Weekly**: Verify backup files exist and are non-empty
- **Automated check script:** [PLACEHOLDER: if you have one]

---

> **Note:** All backup scripts should be stored in a `scripts/` directory and cron job configured on the Pi.
