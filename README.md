# My-Homelab-Setup v4 (AI Assisted)
---
---
# 🏗️ Enterprise-Grade Homelab Architecture & Documentation

Welcome to the central repository for my homelab infrastructure. This documentation details a hybrid-cloud, multi-architecture environment engineered for high availability, secure networking, and automated service orchestration. While built for a lab environment, the design principles closely mirror production-grade DevOps methodologies.

---

## ⚙️ Table of Contents

*   [0. Documentation Index](#0-documentation-index)
*   [1. System Architecture Overview](#1-system-architecture-overview)
*   [2. Compute & Hardware Topology](#2-compute--hardware-topology)
*   [3. Software Stack & DevOps Lifecycle](#3-software-stack--devops-lifecycle)
*   [4. Network Architecture & Security](#4-network-architecture--security)
*   [5. Containerized Services Directory](#5-containerized-services-directory)
*   [6. Engineering Roadmap](#6-engineering-roadmap)
*   [7. Appendices & Administrative Notes](#7-appendices--administrative-notes)

---

## 0. Documentation Index

Detailed documentation is organized into the following directories:

### `network/` — Network Architecture
| Document | Description |
|----------|-------------|
| [`network/topology.md`](network/topology.md) | Physical WAN/LAN topology, overlay networks, traffic flow |
| [`network/ip-assignments.md`](network/ip-assignments.md) | Static IP allocations, WireGuard subnet, Tailscale, DNS |
| [`network/firewall-rules.md`](network/firewall-rules.md) | ER605 load balancing policy, Pi-hole ACLs, container isolation, VPS firewall |

### `services/` — Service Inventory & Operations
| Document | Description |
|----------|-------------|
| [`services/inventory.md`](services/inventory.md) | Full service table: host, port, storage, network namespace |
| [`services/docker-compose-overview.md`](services/docker-compose-overview.md) | Compose stack architecture with representative snippets |
| [`services/container-network-map.md`](services/container-network-map.md) | Network isolation model, port mapping, communication rules |
| [`services/migration-plan.md`](services/migration-plan.md) | Docker → k3s phased migration plan (priority: Nextcloud + Immich) |

### `procedures/` — Operational Runbooks
| Document | Description |
|----------|-------------|
| [`procedures/backup-restore.md`](procedures/backup-restore.md) | Backup schedules, DB dumps, volume backups, restore steps |
| [`procedures/deployment.md`](procedures/deployment.md) | Standard update workflow, rollback, adding new services |
| [`procedures/vpn-config.md`](procedures/vpn-config.md) | WireGuard, Tailscale, and Gluetun configuration |

### `diagrams/` — Visual Reference
| Document | Description |
|----------|-------------|
| [`diagrams/architecture-overview.md`](diagrams/architecture-overview.md) | Narrative walkthrough of the network diagram |
| [`diagrams/Updated Homelab Architecture v3.drawio.png`](diagrams/Updated%20Homelab%20Architecture%20v3.drawio.png) | Visual network topology diagram |

---

## 1. System Architecture Overview

This deployment leverages a hybrid topology, blending local bare-metal edge compute with a remote cloud VPS. The system is designed to achieve low-latency local execution for storage-heavy workloads while maintaining a highly secure, public-facing ingress point via encrypted overlay networks.

---

## 2. Compute & Hardware Topology

### Primary On-Premises Node
*   **Compute:** Raspberry Pi 5 Model B (ARM64 Architecture)
*   **Memory:** 8GB LPDDR4X SDRAM
*   **Storage subsystem:** 6TB total (2×3TB HDDs combined via LVM, mounted at `/mnt/lvm`)

### Auxiliary Cloud Node
*   **Instance Type:** Remote Virtual Private Server (VPS)
*   **Role:** Acts as a secure public edge router, traffic forwarder, and low-overhead Docker service host.

### Planned Cluster Expansion
*   **Target State:** 3-Node High-Availability (HA) Cluster
*   **Architecture:** Heterogeneous multi-architecture compute layer supporting both `ARM64` and `AMD64` target images seamlessly.

---

## 3. Software Stack & DevOps Lifecycle

### Core Operating Systems & Runtimes
| Layer | Component | Deployment Scope |
| :--- | :--- | :--- |
| **Operating System** | Ubuntu Server LTS | Standardized across bare-metal and cloud VPS instances |
| **Container Engine** | Docker Engine & Docker Compose | Current runtime environment for decoupled applications |
| **Orchestration (Target)** | Kubernetes (k3s distribution) | Upcoming migration target for automated scaling and self-healing |

> 💡 **Infrastructure as Code (IaC) Note:** All environment variables, container manifests, and service configurations are maintained inside version-controlled Git repositories. Deployment lifecycles are driven by custom automation scripts and modularized Docker Compose structures.

---

## 4. Network Architecture & Security

### Edge Routing & Redundancy
*   **Hardware Gateway:** TP-Link Omada ER605 Gigabit VPN Router.
*   **WAN Topology:** Dual-ISP Multi-Homing setup configured for active/active load balancing and automatic failover, maximizing throughput and network resilience.
*   **Local Distribution:** Managed Layer 2/3 LAN switching fabric.

### Overlay Networks & Secure Tunnels
*   **WireGuard (Server / Client):** The cloud VPS runs a WireGuard server. All LAN devices (Pi, Garuda Desktop, laptops, phones) connect as WireGuard clients. The VPS routes traffic between peers and provides secure access to services and DNS.
*   **Reverse Proxy:** Caddy runs on the VPS, terminating HTTPS for public subdomains (e.g., `nextcloud.example.com`) and proxying requests through the WireGuard tunnel to the appropriate service on the Pi.
*   **DNS:** Pi-hole runs as a container on the VPS, listening on the WireGuard interface. All LAN devices are configured with `10.9.0.1` as their DNS server, forwarding queries through the tunnel.
*   **Fallback Mesh:** Tailscale integration, functioning as a CGNAT-traversing overlay network for secondary access and out-of-band administration.

---

## 5. Containerized Services Directory

The infrastructure hosts an array of containerized applications segmenting media, storage, security, and AI workloads:

### Application Delivery & Management
*   **Dashy:** Centralized dashboard acting as the unified single-pane-of-glass application portal, hosted on the VPS as a self-contained Docker container with dynamic YAML-based configuration, accessed at `home.aryadivap.com`.
*   **Portainer CE:** Multi-cluster container management platform deployed agentless on both the VPS (at `portainer.aryadivap.com`) and Raspberry Pi (at `portainer-home.aryadivap.com`).

### Storage & Identity Asset Management
*   **Nextcloud Hub:** High-availability self-hosted productivity suite utilized for automated file synchronization and cross-device document backups.
*   **Immich:** High-performance, self-hosted asset management engine utilized for automated mobile photo/video backup.


### Secure Communications
*   **Matrix Synapse:** Self-hosted, end-to-end encrypted chat server for secure private messaging with friends, deployed on the Raspberry Pi 5 with a PostgreSQL 15 Alpine backend.
*   **LiveKit Server:** WebRTC infrastructure enabling high-quality voice and video calls for Element X clients, bundled with Matrix RTC JWT for token-based authentication (port 8081).

### Media & Entertainment Subsystem
*   **Jellyfin:** Open-source media server operating on a strict **read-only** mount isolated from the underlying storage layer, integrated with Nextcloud directories.
*   **The ARR Suite (Prowlarr / Radarr):** Automated media indexer and lifecycle management pipeline.
*   **qBittorrent / Gluetun:** Downstream torrent client strictly bound via network namespaces to a Gluetun VPN container utilizing Surfshark (WireGuard/OpenVPN protocols) to prevent IP leakage.

### Network Security & Observability
*   **Pi-hole:** Network-wide upstream DNS sinkhole hosted on the VPS. All LAN devices route DNS queries through the WireGuard tunnel to Pi-hole for ad-blocking and local DNS resolution.
*   **Caddy:** Automated HTTPS reverse proxy on the VPS, proxying public subdomains to Pi services over the WireGuard tunnel.
*   **Vault Warden:** Self-hosted, privacy-first password manager (Bitwarden-compatible) deployed on the VPS for secure credential storage and cross-device sync, accessed at `vault.aryadivap.com`.
*   **Uptime Kuma:** State-driven HTTP/TCP/Ping monitoring solution hosted on the VPS, providing real-time alerts on container availability and network latencies.

### Core AI & Compute Offloading
*   **Open WebUI:** Front-end orchestration interface hosted separately on an advanced Garuda Arch Linux desktop node. It offloads highly parallelized Large Language Model (LLM) inference processes to a dedicated GPU via the Ollama API runtime.

---

## 6. Engineering Roadmap

*   [ ] **Phase 1: Cluster Consolidation** — Migrate independent nodes into a unified 3-node multi-arch k3s cluster.
*   [ ] **Phase 2: Advanced Ingress & SDN** — Implement reverse proxies (e.g., Traefik or Nginx Proxy Manager) integrated with Let's Encrypt automated ACME certificate renewals.
*   [ ] **Phase 3: Deep Observability** — Roll out a Prometheus and Grafana stack for granular hardware and cluster metrics visualization.

---

## 7. Appendices & Administrative Notes

*   **Sanitization & Security:** In compliance with security best practices, production IPv4/IPv6 addresses, customized port bounds, and private cryptographic keys have been scrubbed from this public specification and replaced with environment variables.
*   **Static Asset Retention:** Media paths and internal documentation references are currently undergoing a consolidation phase; broken links will be resolved as storage volumes stabilize.
*   **Governance:** This documentation is maintained iteratively under version-control principles. Updates are committed upon significant architectural revisions rather than minor configuration drifts.

***
⚡ *Generated and maintained by yours truly.*
