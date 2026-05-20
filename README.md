# My-Homelab-Setup v3
---
---
# 🏗️ Enterprise-Grade Homelab Architecture & Documentation

Welcome to the central repository for my homelab infrastructure. This documentation details a hybrid-cloud, multi-architecture environment engineered for high availability, secure networking, and automated service orchestration. While built for a lab environment, the design principles closely mirror production-grade DevOps methodologies.

---

## ⚙️ Table of Contents

*   [1. System Architecture Overview](#1-system-architecture-overview)
*   [2. Compute & Hardware Topology](#2-compute--hardware-topology)
*   [3. Software Stack & DevOps Lifecycle](#3-software-stack--devops-lifecycle)
*   [4. Network Architecture & Security](#4-network-architecture--security)
*   [5. Containerized Services Directory](#5-containerized-services-directory)
*   [6. Engineering Roadmap](#6-engineering-roadmap)
*   [7. Appendices & Administrative Notes](#7-appendices--administrative-notes)

---

## 1. System Architecture Overview

This deployment leverages a hybrid topology, blending local bare-metal edge compute with a remote cloud VPS. The system is designed to achieve low-latency local execution for storage-heavy workloads while maintaining a highly secure, public-facing ingress point via encrypted overlay networks.

---

## 2. Compute & Hardware Topology

### Primary On-Premises Node
*   **Compute:** Raspberry Pi 5 Model B (ARM64 Architecture)
*   **Memory:** 8GB LPDDR4X SDRAM
*   **Storage subsystem:** 6TB High-Capacity Mechanical HDD (Provisioned for centralized network-attached storage)

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
*   **Primary Ingress/Egress:** WireGuard Peer-to-Peer (P2P) point-to-point encrypted tunnels, routing traffic securely between the on-premises node and the public VPS cloud boundary.
*   **Fallback Mesh:** Tailscale integration, functioning as a CGNAT-traversing overlay network for secondary access and out-of-band administration.

---

## 5. Containerized Services Directory

The infrastructure hosts an array of containerized applications segmenting media, storage, security, and AI workloads:

### Application Delivery & Management
*   **Homepage:** Distributed dashboard acting as the unified single-pane-of-glass application portal (`gethomepage.dev`).
*   **Portainer CE:** Multi-cluster container management platform deployed agentless to oversee workloads across both local and VPS engines.

### Storage & Identity Asset Management
*   **Nextcloud Hub:** High-availability self-hosted productivity suite utilized for automated file synchronization and cross-device document backups.
*   **Immich:** High-performance, self-hosted asset management engine utilized for automated mobile photo/video backup.

### Media & Entertainment Subsystem
*   **Jellyfin:** Open-source media server operating on a strict **read-only** mount isolated from the underlying storage layer, integrated with Nextcloud directories.
*   **The ARR Suite (Prowlarr / Radarr):** Automated media indexer and lifecycle management pipeline.
*   **qBittorrent / Gluetun:** Downstream torrent client strictly bound via network namespaces to a Gluetun VPN container utilizing Surfshark (WireGuard/OpenVPN protocols) to prevent IP leakage.

### Network Security & Observability
*   **Pi-hole:** Network-wide upstream DNS sinkhole utilized for ad-blocking and local DNS split-horizon routing.
*   **Uptime Kuma:** State-driven HTTP/TCP/Ping monitoring solution providing real-time alerts on container availability and network latencies.

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
