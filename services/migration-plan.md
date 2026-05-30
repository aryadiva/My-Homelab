# Migration Plan: Docker Compose → k3s

## Overview

This document outlines the phased migration from the current Docker Compose-based deployment to a 3-node multi-architecture k3s Kubernetes cluster. The migration prioritizes data-heavy, stateful services first.

**Target State:** 3-node HA cluster (ARM64 + AMD64)

---

## Phase 1 — Foundation (Pre-Migration)

| Task | Details | Status |
|------|---------|--------|
| Provision k3s nodes | 3 nodes: [PLACEHOLDER: node specs and hostnames] | [ ] |
| Install k3s | Multi-arch cluster with ARM64/AMD64 support | [ ] |
| Set up storage class | Longhorn or local-path provisioner for PVCs | [ ] |
| Set up ingress controller | Traefik (bundled with k3s) or Nginx Proxy Manager | [ ] |
| Set up cert-manager | Let's Encrypt automated ACME certificates | [ ] |
| Set up metallb | Load balancer IP pool (or use Traefik HostPort) | [ ] |

---

## Phase 2 — Priority Migration: Nextcloud + Immich

These two services hold the most critical user data (file sync + photo library). They should be migrated first to benefit from k3s resilience.

### 2.1 Nextcloud

**Current State (Compose):**
- Web server + PostgreSQL + Redis on Docker
- Data: `/mnt/lvm/nextcloud/data`
- DB: `./nextcloud/db` (PostgreSQL volume)
- Config: `./nextcloud/config`
- Apps: `./nextcloud/custom_apps`

**Migration Steps:**

1. **Pre-migration:**
   - Perform full backup (see [`../procedures/backup-restore.md`](../procedures/backup-restore.md))
   - Run `occ maintenance:mode --on`
   - Dump PostgreSQL: `pg_dump -U nextcloud nextcloud > nextcloud-dump.sql`

2. **k3s Deployment:**
   - Create PVCs for data, config, apps (point to the same underlying NFS/Longhorn storage as `/mnt/lvm`)
   - Deploy `StatefulSet` for Nextcloud with:
     - `postgres:15-alpine` as a dependent `StatefulSet`
     - `redis:7-alpine` as a `Deployment`
     - Ingress rule for `nextcloud.[PLACEHOLDER: domain]`
   - ConfigMap for `config.php` and environment variables

3. **Data Migration:**
   - Copy data files to the new PVC-backed storage
   - Restore PostgreSQL dump into the k3s Postgres pod
   - Re-attach external volume if using NFS-backed PVC (`/mnt/lvm/nextcloud/data` → PV)

4. **Validation:**
   - Verify file sync works
   - Verify all apps are enabled
   - Take Nextcloud out of maintenance mode: `occ maintenance:mode --off`

**Helm Chart Reference:** [nextcloud/all-in-one](https://github.com/nextcloud/all-in-one) or manual manifests.

### 2.2 Immich

**Current State (Compose):**
- `immich-server`, `immich-microservices`, `immich-machine-learning`
- Data: `/mnt/lvm/immich/photos`
- DB: `./immich/db` (PostgreSQL)
- Config: `./immich/config`
- Redis

**Migration Steps:**

1. **Pre-migration:**
   - Full backup of DB and upload directory
   - Dump Immich PostgreSQL: `pg_dump -U immich immich > immich-dump.sql`

2. **k3s Deployment:**
   - Use the official [Immich k8s Helm chart](https://immich.app/docs/administration/kubernetes)
   - Create PVCs for upload, config, DB
   - Deploy microservices as separate Deployments
   - Ingress rule for `immich.[PLACEHOLDER: domain]`

3. **Data Migration:**
   - Copy `upload/` directory to PVC
   - Restore PostgreSQL dump
   - Machine learning model weights will be re-downloaded automatically

4. **Validation:**
   - Verify photo upload and thumbnail generation
   - Verify ML facial recognition and search work
   - Check mobile app sync

---

## Phase 3 — Secondary Services

| Service | Migration Priority | Method | Notes |
|---------|-------------------|--------|-------|
| Jellyfin | Medium | Deployment + PVC | Read-only media mount; keep `/mnt/lvm/media` as NFS PV |
| Prowlarr | Medium | Deployment + PVC | Stateless config; easy to redeploy |
| Radarr | Medium | Deployment + PVC | Depends on download path from qBittorrent |
| qBittorrent + Gluetun | Low | Pod with sidecar (Gluetun) | Must keep VPN isolation model. k3s equivalent: Gluetun as init container or sidecar with `hostNetwork: true` + NET_ADMIN |
| Pi-hole | Low | Deployment + hostNetwork | Needs host port 53; use DaemonSet or hostNetwork binding |
| Homepage | Low | Deployment + ConfigMap | Stateless; easy to redeploy |
| Portainer | Consider dropping | N/A | k3s has built-in UI (Dashboard); Portainer no longer needed |
| Uptime Kuma | Low | Deployment + PVC | Replace with Prometheus/Grafana in future (Roadmap Phase 3) |
| Open WebUI + Ollama | Stays on Garuda | N/A | GPU passthrough to k3s is complex; keep on dedicated desktop |

### Gluetun VPN Sidecar Pattern for k3s

```yaml
# Conceptual pod template for qBittorrent + Gluetun sidecar
apiVersion: apps/v1
kind: Deployment
metadata:
  name: qbittorrent
spec:
  template:
    spec:
      initContainers:
        - name: gluetun-init
          image: qmcgaw/gluetun:latest
          securityContext:
            capabilities:
              add: ["NET_ADMIN"]
          env:
            - name: VPN_SERVICE_PROVIDER
              value: surfshark
            # ... other VPN env vars
      containers:
        - name: qbittorrent
          image: lscr.io/linuxserver/qbittorrent:latest
          # Shares the network namespace of the pod (which includes gluetun)
```

---

## Phase 4 — Post-Migration Cleanup

| Task | Details |
|------|---------|
| Decommission Compose services | Stop old Docker containers |
| Repurpose Docker volumes | Reclaim or archive old volume paths |
| Update DNS | Point service domains to new k3s ingress IPs |
| Document new architecture | Update this document and architecture diagram |

---

## Rollback Plan

If a migration fails:

1. Stop k3s pods for the migrated service
2. Restore Docker Compose service from backup
3. Restore PostgreSQL dump into the Docker DB volume
4. Point DNS back to old Docker host IP
5. Investigate failure cause before retrying

---

> **Note:** This plan assumes a shared-NFS or Longhorn storage layer accessible by both the Docker host and k3s nodes during migration. Plan your storage class accordingly.
