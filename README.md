# Proxmox / Docker Media Stack: Gluetun VPN & *Arr Automation

This repository contains a unified `docker-compose.yml` deployment for a complete, production-ready media automation stack running on Docker / Proxmox VE. 

All torrent download traffic is routed strictly through a **Gluetun WireGuard VPN client container**, while maintaining seamless local communication between services (Radarr, Sonarr, Lidarr, Bazarr, Prowlarr, Jellyfin). The file system architecture strictly follows **[TRaSH Guides](https://trash-guides.info/)** standards to ensure instant, zero-cost hardlinks and atomic file moves.

---

## 🏗️ Architecture Overview

```text
                                  [ Internet / Tracker Network ]
                                                ▲
                                                │ (WireGuard Tunnel)
                                                ▼
┌───────────────────────────────────────────────────────────────────────────┐
│  GLUETUN CONTAINER (VPN Client & Firewall)                                │
│                                                                           │
│   Shared Network Namespace:                                               │
│   ├── Port Mapping: 8080 (Exposes qBittorrent WebUI)                      │
│   └── qBittorrent Container (network_mode: "service:gluetun")             │
└───────────────────────▲───────────────────────────────────────────────────┘
                        │
                        │ (Internal Docker Network: `arr_network`)
                        │
  ┌─────────────────────┼─────────────────────┬─────────────────────┐
  │                     │                     │                     │
┌─┴──────────────┐   ┌──┴─────────────┐    ┌──┴─────────────┐    ┌──┴─────────────┐
│    RADARR      │   │     SONARR     │    │   PROWLARR     │    │   JELLYFIN     │
│ (Port 7878)    │   │  (Port 8989)   │    │ (Port 9696)    │    │ (Port 8096)    │
└────────────────┘   └────────────────┘    └────────────────┘    └────────────────┘
```

---

## 📂 Storage Architecture (TRaSH Guides Standard)

To allow instant **Atomic Moves** and **Hardlinks** (preventing slow, disk-heavy duplicate copies across drives), all containers share a single root mount `/data`.

### Host Directory Layout (`/data`)

Create this folder hierarchy on your host machine before starting the stack:

```text
/data
├── torrents
│   ├── books
│   ├── movies
│   ├── music
│   └── tv
└── media
    ├── books
    ├── movies
    ├── music
    └── tv
```

### Volume Mapping Principles

Every media-handling service (`qbittorrent`, `radarr`, `sonarr`, `lidarr`, `bazarr`, `jellyfin`) must mount the exact same root path:

```yaml
volumes:
  - /data:/data
```

> ⚠️ **CRITICAL TRaSH RULE:** Never map individual subfolders like `/data/media:/media` or `/data/torrents:/downloads`. Subfolder volume mounts break Linux hardlinking across mounts and force Docker to perform full disk copies.

#### Service Category Paths:
* **qBittorrent Categories:**
  * Movies category save path: `/data/torrents/movies`
  * TV category save path: `/data/torrents/tv`
* **Arr App Root Folders:**
  * Radarr Root Folder: `/data/media/movies`
  * Sonarr Root Folder: `/data/media/tv`

---

## 🔒 Network Routing & VPN Isolation

1. **qBittorrent Traffic Isolation:**  
   `qbittorrent` uses `network_mode: "service:gluetun"`. This forces qBittorrent into Gluetun's network namespace. It has no independent network interface; all inbound/outbound P2P traffic **must** pass through Gluetun's WireGuard tunnel. If Gluetun loses its connection, qBittorrent's connection drops immediately (functioning as an automatic kill-switch).

2. **Accessing qBittorrent's Web UI:**  
   Because qBittorrent inherits Gluetun's network stack, port `8080` is exposed to the local network via **Gluetun's `ports:` block**, not qBittorrent's.

3. **Service-to-Service API Communication:**  
   When connecting download clients inside Prowlarr, Radarr, or Sonarr, you must target the host name **`gluetun`** on port **`8080`**.

---

## ⚠️ Critical Setup Instructions

### 1. Disable qBittorrent Host Header Validation
By default, official and Hotio qBittorrent images enforce **Host Header Validation**. When applications like Prowlarr attempt to send API requests to qBittorrent via the Docker container name (`http://gluetun:8080`), qBittorrent drops the request with a `400 Bad Request` or connection refused error.

To allow Prowlarr and the *Arr stack to communicate with qBittorrent, append the following setting to your qBittorrent configuration file at `/docker/appdata/qbittorrent/config/qBittorrent.conf`:

```ini
[Preferences]
WebUI\HostHeaderValidation=false
```

**One-line command to apply on host:**
```bash
echo 'WebUI\HostHeaderValidation=false' >> /docker/appdata/qbittorrent/config/qBittorrent.conf
docker compose restart qbittorrent
```

---

### 2. Service Integration Reference

| Application | Config Field | Setting / Value | Notes |
| :--- | :--- | :--- | :--- |
| **Prowlarr** | **Download Client Host** | `gluetun` | Directs API requests to the VPN network namespace |
| **Prowlarr** | **Port** | `8080` | Web UI / API port mapped in Gluetun |
| **Radarr / Sonarr** | **Download Client Host** | `gluetun` | Same as Prowlarr |
| **Radarr / Sonarr** | **Port** | `8080` | |
| **Prowlarr** | **Radarr Integration** | `http://radarr:7878` | Resolves over internal `arr_network` |
| **Prowlarr** | **Sonarr Integration** | `http://sonarr:8989` | Resolves over internal `arr_network` |

---

## 🧪 Post-Deployment Verification

1. **P2P Torrent Leak Test:**
   * Open qBittorrent at `http://<HOST_IP>:8080`.
   * Go to [IPLeak.net](https://ipleak.net) and copy a **Torrent Address Detection** magnet link.
   * Add the link to qBittorrent and verify that the active IP matches your NordVPN WireGuard IP, not your home ISP.

2. **Test Download Client in Prowlarr:**
   * Go to **Prowlarr** $\rightarrow$ **Settings** $\rightarrow$ **Download Clients** $\rightarrow$ **qBittorrent**.
   * Set **Host** to `gluetun`, **Port** to `8080`, enter credentials, and click **Test**.

3. **Confirm TRaSH Hardlinks:**
   * Trigger a download from Radarr or Sonarr. Once complete, check `/data/media/movies/`. The file should move instantly without delaying or duplicating disk usage.
