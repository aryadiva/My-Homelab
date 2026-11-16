# Backup & Restore — Blueprint & Reference (Synced 2026-11-16)

> ⚠️ **Status:** This is a reference blueprint. Some services (Nextcloud, Immich) have been backed up manually in the past; automated schedules are TBD.  
> All backup scripts and schedules marked as `TO-IMPLEMENT` until scheduled cron jobs are created.

---

## Current Manual Backups (Known Past Behavior)

The user has described performing manual database dumps and file backups before Nextcloud upgrades (version 32.x → 33 exploration). See knowledge base for historical context.  
Automated tiered backup still not in production.

---

## Planned Backup Strategy (To Implement)

| Tier | Scope | Frequency | Method |
|------|-------|-----------|--------|
| **Tier 1** | PostgreSQL (Nextcloud, Immich) database dumps | Daily at midnight | `pgdump` → gzip |
| **Tier 2** | Application data (`/mnt/lvm/*`) — files & DB volumes if not dumped | Daily | tar to `/mnt/lvm/backups/volumes` or external drive |
| **Tier 3** | Config directories | Weekly (Sunday) | zip backup of config folders, restore tested quarterly |
| **Tier 4**: Docker Compose configs | Version control via Git commits | Every change | Already tracked in repo |
| **Tier 5** | Off-site / Cloud replication | Monthly | rclone → Backblaze B2 or S3-compatible storage |
| **Tier 6** | Full state snapshot (volumes + compose config) | Quarterly — before major upgrades | Snapshot + `docker compose config > configs.yml` |

---

## PostgreSQL Dumps — Reference Script

```bash
#!/bin/bash
# backup-postgres.sh — To be scheduled in `/etc/cron.daily/`
BACKUP_DIR="/mnt/lvm/backups/postgres"
DATE=$(date +%Y%m%d_%H%M%S)

docker exec nextcloud-db pg_dump -U nextcloud nextcloud > "$BACKUP_DIR/nextcloud_$DATE.sql" || exit 1
docker exec immich-db pg_dump -U immich immich     > "$BACKUP_DIR/immich_    $DATE.sql" || exit 1

gzip "$BACKUP_DIR"/nextcloud_$DATE.sql.gz && gzip "$BACKUP_DIR"/immich_$DATE.sql.gz

# Retain last 7 days
find "$BACKUP_DIR" -name "*.sql.gz" -mtime +7 -delete
```

> ⚠️ Immich DB engine is historically thought to be PostgreSQL 15 Alpine but is not confirmed — verify with `docker exec immich-db psql -l` before scripting dumps.

---

## Data Volume Backups (Reference)

```bash
#!/bin/bash
# backup-volumes.sh — To be scheduled
BACKUP_DIR="/mnt/lvm/backups/volumes"
DATE=$(date +%Y%m%d)

tar -czf "$BACKUP_DIR/nextcloud_data-$DATE.tar.gz"  --exclude='*.tmp' -C /mnt/lvm/nextcloud data
tar -czf "$BACKUP_DIR/immich_photos-$DATE.tar.gz"   -C /mnt/lvm/immich photos

# Config-only weekly tar (Prowlarr, Radarr, qBittorrent etc.)
tar -czf "$BACKUP_DIR/configs-$DATE.tar.gz" ./prowlarr ./radarr ./qbittorrent ./jellyfin
```

> Retention policy: keep volumes 14 days, configs 30 days. Adjust based on user requirements.

---

## Restore Procedures — Verified by User

### Nextcloud (MariaDB + Data)

1. Stop all Nextcloud containers (`docker compose down nextcloud`, or enable maintenance mode first) to prevent data races during restore.
2. Restore database:
   ```bash
   gzip -dc <backup-file>.sql.gz | docker exec -i nextcloud-db mariadb -u root -p [PLACEHOLDER_ROOT_PASSWORD] nextcloud
   ```
   Adjust user/password/DB name; do NOT store real credentials in this doc.
3. Restore data files to `/mnt/lvm/nextcloud-data` — replace directory permissions (www-data +).

### Immich Upload / DB — Reference Flow

1. Before restoring, confirm DB type and engine:
   ```bash
   docker exec immich-db psql -d postgresql_database_name 2>/dev/null || echo "not PostgreSQL"
   ```
2. Stop immich-db container to prevent concurrent writes during restore.
3. Restore database with appropriate tool (PostgreSQL or MariaDB if engine differs) — refer to backup scripts.
4. Restore upload directory at `/mnt/lvm/immich-app` or legacy `/mnt/lvm/immich/photos`.

See `procedures/deployment.md` for generic container update workflows that cover rollback.

---

## External Backups — TBD

Options:
1. Backblaze B2 — S3-compatible, low cost (~$5/TB/month)
2. rsync.net — flat-rate storage
3. rclone to home server VPS or public cloud bucket
4. Physical USB backup drive for quarterly full volume snapshots

---
