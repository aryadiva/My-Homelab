# My-Homelab-Setup
---
---

# 🏠 Updated Homelab Architecture & Services

This documentation covers the expanded containerized ecosystem running on my Ubuntu Server, focused on network-wide ad-blocking, infrastructure monitoring, and automated media management. See diagrams for more visual clarity.

## 📡 Updated Network Topology
The core networking is managed by the **Omada Load Balancer** and **Deco Mesh**, but the internal service layer now includes a dedicated "Privacy Tunnel" for specific traffic.

## 🏗️ Expanded Server Architecture (`192.168.10.188`)

My Raspberry Pi / Ubuntu Server now hosts an advanced Docker stack managed via **Portainer**. The stack is divided into three functional pillars:

### 1. Management & Health
* **Portainer:** My centralized GUI for container orchestration. It allows me to monitor resource usage, manage persistent volumes, and quickly deploy stack updates via Docker Compose.
* **Uptime Kuma:** The "Watchtower" of the lab. It monitors all local services (Pi-hole, Jellyfin, etc.) and external pings, providing a real-time status dashboard and downtime alerts.

### 2. Network Security & Privacy
* **Pi-hole:** Acts as the primary DNS sinkhole for the entire `192.168.10.x` and `IoT` zones. It blocks trackers and advertisements at the DNS level before they reach any device.
* **Gluetun:** My dedicated VPN client container. It creates a secure, encrypted tunnel. I route high-traffic services through this container to ensure that their external metadata is never exposed to the ISP. I used my Surfshark VPN subscription for this.

### 3. Automated Media Pipeline (The "Arr" Stack)
I have implemented a fully automated lifecycle for media management. These containers communicate over a private Docker network and relies on gluetun to provide secure VPN connection:

| Service | Role | Workflow |
| :--- | :--- | :--- |
| **Prowlarr** | Indexer Manager | Synchronizes search "trackers" across the entire stack. |
| **Radarr** | Movie Automator | Monitors for new releases and manages the movie library metadata. |
| **qBittorrent** | Download Engine | **Routed through Gluetun.** Handles the actual data transfer over VPN. |
| **Jellyfin** | Media Server | The frontend UI that streams the finished library to my devices. |

---

## 🛠️ Traffic Flow: The "Privacy Tunnel"

A critical feature of this updated architecture is the **Network Routing**:

1.  **Request:** Radarr identifies a missing movie.
2.  **Search:** Prowlarr finds a source.
3.  **Transfer:** Radarr sends the "magnet" to **qBittorrent**.
4.  **Encryption:** qBittorrent is configured to use **Gluetun** as its network stack (`network_mode: service:gluetun`). 
5.  **Exit:** All download traffic leaves my home network encrypted via the VPN, while the rest of my LAN (like a Zoom call on my Desktop) remains on the high-speed ISP line.

## 🔒 Access Control
All services mentioned above are accessible remotely via the **Tailscale** mesh VPN. No ports (80, 443, etc.) are opened on the Omada Load Balancer, maintaining a "stealth" profile from the public internet.

---
