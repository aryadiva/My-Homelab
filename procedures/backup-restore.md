# Backup & Restore Procedures — Future Plan

## ⚠️ Current Status: **No Active Backup Plan yet!**

This document serves as a **blueprint for a future backup strategy**. No automated backups, cron jobs, or off-site replication are currently configured. The scripts and schedules below are **planned work items** — nothing is running in production yet.

**Priority:** High — data loss risk exists for Nextcloud (files + DB) and Immich (photos + DB).

---

## 1. Planned Backup Tiers

| Tier | Scope | Target Frequency | Recommended Method |
|------|-------|-----------------|-------------------|
| **Tier 1** | DB dumps (Nextcloud, Immich PostgreSQL) | Daily | `pg_dump` → compressed archive |
| **Tier 2** | Application data (Nextcloud files, Immich photos) | Daily | rsync / tar to backup volume |
| **Tier 3** | Config dirs (Prowlarr, Radarr, Jellyfin, qBittorrent, etc.) | Weekly | tar → backup volume |
| **Tier 4** | Docker Compose + Git-tracked files | Per-change | Git commit (already done) |
| **Tier 5** | Off-site / cloud replication | Monthly | Backblaze B2, rsync.net, or S3-compatible |
| **Tier 6** | Full system state | Monthly | Volume snapshot + `docker-compose config` |

> ⬜ **TODO:** Decide on a backup storage path (e.g., `/mnt/lvm/backups/` on the LVM volume, or a dedicated external drive).

---

## 2. Database Dumps (Planned)

### PostgreSQL (Nextcloud, Immich)

The script below is a **template** for future implementation. It is not yet scheduled or deployed.

```bash
#!/bin/bash
# backup-postgres.sh (NOT YET ACTIVE)
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

## 3. Volume Backup (Planned)

### Application Data Volumes

```bash
#!/bin/bash
# backup-volumes.sh (NOT YET ACTIVE)
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

## 4. Restore Procedures (Reference)

> These restore procedures are **documented for future use** — they cannot be run yet since no backups exist.

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

## 5. Off-Site Backup (Planned)

> ⬜ **TODO:** Choose and configure an off-site backup destination.

| Option | Pros | Cons | Est. Cost |
|--------|------|------|-----------|
| **Backblaze B2** | S3-compatible, low cost | Egress fees on restore | ~$5/TB/month |
| **rsync.net** | Flat-rate storage | Slightly higher cost | ~$20/TB/month |
| **Rclone + S3** | Flexible, self-managed | Requires S3 provider | Varies |
| **External HDD (cold)** | One-time cost, no monthly fee | Manual, no automation | $50–100 |

**Recommended approach:** Start with local backups to `/mnt/lvm/backups/`, then add off-site replication via `rclone` once the local pipeline is stable.

---

## 6. Implementation Roadmap

| # | Task | Dependencies | Est. Effort |
|---|------|-------------|-------------|
| 1 | Create backup directory: `mkdir -p /mnt/lvm/backups/{postgres,volumes}` | LVM mounted | 5 min |
| 2 | Place scripts in `~/scripts/` and test manually | Step 1 | 30 min |
| 3 | Schedule cron jobs for DB + volume dumps | Step 2 | 15 min |
| 4 | Add Pi-hole + VPS backup via SSH | SSH keys configured | 30 min |
| 5 | Configure off-site sync (rclone to B2, etc.) | Step 3 | 1 hr |
| 6 | Document and test restore procedure | Steps 1–5 | 1 hr |
| 7 | Set up monthly backup verification | Step 6 | 30 min |

---

## 7. Backup Verification (Planned)

- **Monthly:** Test-restore Nextcloud + Immich to a staging directory.
- **Weekly:** Verify backup files exist and are non-empty.
- **Automated check script:** ⬜ TODO — write a `verify-backups.sh` script.

---

> **Note:** All backup scripts should be stored in a `scripts/` directory once implemented. Cron jobs should be configured on the Pi.
