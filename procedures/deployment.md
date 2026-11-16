# Deployment Guide — Services Update & Rollback (Synced 2026-11-16)

> This document covers standard container update, rollback, and adding new services for Docker Compose deployments.  
> All steps assume working knowledge of the host's `docker compose` context directory (`./infra.yml`, `./media.yml` as in docker-compose-overview.md).

---

## Updating a Service (Rolling Deployment)

### 1. Pull Latest Image

```bash
cd /path/to/compose/directory
docker compose pull [service-name]
```

Example: `docker compose pull nextcloud` — only pulls changed images, not all services (unless no args).

---

### 2. Apply and Restart

```bash
# Apply config and restart the updated service(s)
docker compose up -d [service-name-or-new-compose-file-snapshot]
```

Use `-d` for detached background mode.

---

### 3. Monitor Logs on Startup

```bash
docker compose logs -f [service-name] --tail=30
```

If errors appear at startup, check: image tag compatibility with existing volumes and env vars.

---

## Rolling Back a Service — Known Stable Tags

Nextcloud was rolled back previously from version 32.x after AIO exploration (v33). Current tag unknown.

### Quick Rollback Steps

1. Pin to previous working image in Compose:
   ```yaml
   nextcloud-image: nextcloud:v32-apache  # Replace with actual working tag
   ```

2. Pull and restart:
   ```bash
   docker compose pull nextcloud && docker compose up -d nextcloud
   ```

3. If service fails, switch back to latest for debugging, then re-try previous tag.

---

## Adding a New Container Service

1. Add service block to the appropriate Compose file (`infra.yml`, `media.yml`, or custom one)
2. Ensure network membership (`networks: [media_net]` or your own networks)
3. Pick a non-conflicting port binding (verify against [`services/inventory.md`](../services/inventory.md))
4. Configure volumes for persistent data if required
5. Add to `.env` only secrets and env vars needed
6. Commit any config changes via Git if tracked in repo

Then deploy:

```bash
docker compose up -d new-service-name
docker ps | grep new-service-name
```

Update `services/inventory.md` with the new entry — port, paths, network.

---

## Secret & Environment Variable Management

Sensitive values (DB passwords, API keys, WireGuard private keys, virtual keys) should NOT be committed to Git:

1. Place in `.env` file in the Docker compose directory
2. Add `.env` to `.gitignore` to prevent commits
3. When pulling `docker compose up`, Docker reads local env file and injects secrets at runtime

Example `.env` snippet:

```bash
NEXTCLOUD_DB_PASSWORD=[PLACEHOLDER]    # DO NOT store actual password in docs
IMMICH_DB_PASSWORD=[PLACEHOLDER]       # Replace with a unique strong value on each host
PIHOLE_WEBPASSWORD=[PLACEHOLDER]       # Dashy, Vaultwarden passwords similarly managed
WIREGUARD_PRIVATE_KEY=GF...             # NEVER copy real keys from this doc — use per-host generated keys
```

Verify secrets never appear in Git logs: `git log --all --full-history -- '*.env'` should return no matches on tracked files.

---

## Docker Compose Project Snapshots

When backing up or migrating configs, snapshot compose projects with command:

```bash
# Outputs rendered config YAML to a file for archival
docker compose config > ../archive-compose-config.yml.bak
```

This helps recover if the repo `.yml` files get corrupted but volumes remain healthy.
