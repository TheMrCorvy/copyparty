# copyparty — Homelab File Server

A self-hosted, lightweight file server powered by [copyparty](https://github.com/9001/copyparty), running in Docker. Exposes 3 drives over your local network with no password required. Designed for a progressive deployment path:

**Windows 11 (testing) → macOS Sequoia (homelab) → Ubuntu Server 24.04 (future)**

---

## Project Structure

```
copyparty/
├── config/
│   └── copyparty.conf       ← Share definitions, permissions, global settings
├── k8s/                     ← Future Kubernetes manifests
│   ├── namespace.yaml
│   ├── configmap.yaml       ← Config stored as K8s native resource
│   ├── persistent-volumes.yaml
│   ├── persistent-volume-claims.yaml
│   ├── deployment.yaml
│   └── service.yaml         ← NodePort for LAN access
├── .env                     ← Active drive paths (not committed to git)
├── .env.example             ← Template — copy and edit for each machine
├── .gitignore
├── docker-compose.yml
└── README.md
```

---

## Quick Start

### Prerequisites

- [Docker Desktop](https://www.docker.com/products/docker-desktop/) (Windows / macOS) or Docker Engine (Ubuntu)
- Docker Compose v2+ (bundled with Docker Desktop)

### 1 — Set your drive paths

Edit `.env` with the real paths on your current machine. See the platform guides below for OS-specific path formats.

### 2 — Start the container

```bash
docker compose up -d
```

Docker pulls the image automatically on first run (~50 MB).

### 3 — Open in your browser

```
http://localhost:3923
```

From another device on the same WiFi network:

```
http://<your-machine-ip>:3923
```

### 4 — Stop

```bash
docker compose down
```

---

## Configuration

### Drives & Volumes

Edit `config/copyparty.conf` to change which drives are shared and their names in the browser.

The **container paths** (`/drives/main`, etc.) are fixed — they must match both `copyparty.conf` and `docker-compose.yml`. The **host paths** come from `.env` — this is the only file you change when switching machines.

| `.env` variable   | Container path      | Browser URL  |
| ----------------- | ------------------- | ------------ |
| `MAIN_DRIVE_PATH` | `/drives/main`      | `/main`      |
| `EXT_DRIVE1_PATH` | `/drives/external1` | `/external1` |
| `EXT_DRIVE2_PATH` | `/drives/external2` | `/external2` |

### Permissions

Current setting: **full anonymous access** (`rwmd: *`) — suitable for local-only use.

| Code | Meaning         |
| ---- | --------------- |
| `r`  | Read / download |
| `w`  | Write / upload  |
| `m`  | Move / rename   |
| `d`  | Delete          |

To make a drive read-only, change `rwmd: *` → `r: *` in `copyparty.conf`.

### Reload config without restarting the container

```bash
docker kill --signal=USR1 copyparty
```

---

## Resource Limits

Configured conservatively for a Mac Mini running multiple apps. Tune in `.env`:

| Setting     | Default | Notes                                          |
| ----------- | ------- | ---------------------------------------------- |
| `CPU_LIMIT` | `0.50`  | 50% of one core max                            |
| `MEM_LIMIT` | `256M`  | Increase to `512M` if indexing large libraries |

---

## Deployment Guide

---

### Platform 1: Windows 11

#### Prerequisites

- [Docker Desktop for Windows](https://www.docker.com/products/docker-desktop/)

#### Step 1 — Allow Docker to access your drives

Docker Desktop on Windows has an explicit file-sharing whitelist. Without this, volume mounts appear empty in the container.

> **Docker Desktop → Settings → Resources → File Sharing**

Add the root of every drive you want to share:

- `C:\`
- `D:\`
- `E:\`

Click **Apply & Restart**.

#### Step 2 — Edit `.env`

Use forward slashes (Docker Desktop on Windows translates them automatically):

```dotenv
MAIN_DRIVE_PATH=C:/Users/gonza/Documents
EXT_DRIVE1_PATH=D:/
EXT_DRIVE2_PATH=E:/
TZ=America/New_York
```

#### Step 3 — Start

```powershell
docker compose up -d
```

Browse to `http://localhost:3923`.

#### Finding your Windows IP for LAN access

```powershell
ipconfig
```

Look for the **WiFi adapter → IPv4 Address** (e.g. `192.168.1.xx`). Other devices on the network access copyparty at `http://192.168.1.xx:3923`.

---

### Platform 2: macOS

> **Intel-specific note:** The `ghcr.io/9001/copyparty-ac:latest` image is multi-arch. On Intel it pulls `linux/amd64` automatically — no Rosetta or manual flag needed.

#### Choosing a Docker runtime — Docker Desktop vs OrbStack

Your Mac Mini is an Intel i5 **already running many apps**. Docker Desktop for Mac runs a full Linux VM in the background and consumes **~500 MB RAM** at idle. This is significant overhead on a shared homelab server.

|                                | Docker Desktop                                                | OrbStack (recommended)       |
| ------------------------------ | ------------------------------------------------------------- | ---------------------------- |
| Idle RAM                       | ~500 MB                                                       | ~50–100 MB                   |
| Startup time                   | ~30 seconds                                                   | ~2 seconds                   |
| `docker compose` compatibility | ✅ Full                                                       | ✅ Full — identical commands |
| Cost                           | Free (personal)                                               | Free (personal)              |
| Install                        | [docker.com](https://www.docker.com/products/docker-desktop/) | `brew install orbstack`      |

Both work with this project without any changes to `docker-compose.yml`. OrbStack is strongly recommended for the Mac Mini.

**Install OrbStack:**

```bash
brew install orbstack
```

Then launch OrbStack from Applications. It runs as a lightweight menu-bar app with no extra configuration needed.

#### Step 1 — File Sharing (no manual setup needed)

Unlike Windows, macOS Docker runtimes automatically share the following paths with containers:

- `/Users` — your home folder
- `/Volumes` — **all mounted drives**, including USB externals
- `/private`, `/tmp`

Your external drives mount under `/Volumes/` automatically when plugged in, so **no whitelist configuration is needed**.

> **One exception:** If macOS mounts a drive at a path outside `/Volumes` (rare), add it manually in Docker Desktop → Settings → Resources → File Sharing. OrbStack handles any path automatically without configuration.

#### Step 2 — Grant Full Disk Access (macOS Sequoia requirement)

macOS Sequoia's privacy system can block Docker from reading external volumes even when they appear mounted. This causes drives to show up in the browser but appear empty inside.

**For Docker Desktop:**

> **System Settings → Privacy & Security → Full Disk Access → toggle on Docker Desktop or Orbstack**

This is a **one-time step** that survives reboots and updates.

#### Step 3 — Enable VirtioFS for faster file I/O

_(Docker Desktop only — OrbStack uses a fast native layer by default and does not need this)_

VirtioFS is a modern, significantly faster filesystem sharing protocol versus the legacy `osxfs`. On file-heavy workloads (large transfers, directory scans) the difference is very noticeable on Intel hardware.

> **Docker Desktop → Settings → General → "Use VirtioFS for file sharing"** → enable → **Apply & Restart**

#### Step 4 — Find your drive names

External drives on macOS appear under `/Volumes/` with the name shown in Finder. Run this in Terminal to see exactly what's mounted:

```bash
ls /Volumes/
```

Example output:

```
Macintosh HD    ExternalDrive1    ExternalDrive2
```

Use these exact names in your `.env` in the next step.

#### Step 5 — Edit `.env`

```dotenv
CONFIG_PATH=./config

# Use the exact names from: ls /Volumes/
MAIN_DRIVE_PATH=/Users/gonza/Documents         # or a folder on the internal drive
EXT_DRIVE1_PATH=/Volumes/ExternalDrive1
EXT_DRIVE2_PATH=/Volumes/ExternalDrive2

TZ=America/New_York    # update to your actual timezone
CPU_LIMIT=0.50
MEM_LIMIT=256M
```

> **Drive name with spaces?** Write it as-is in `.env` without quotes:
> `EXT_DRIVE1_PATH=/Volumes/My External Drive`

#### Step 6 — Start

```bash
docker compose up -d
```

Browse to `http://localhost:3923`.

#### Step 7 — Auto-start on boot

**OrbStack:** Enabled by default. OrbStack starts at login and automatically restarts containers marked `restart: unless-stopped`.

**Docker Desktop:** Go to Docker Desktop → Settings → General → enable **"Start Docker Desktop when you log in"**. The `restart: unless-stopped` policy handles restarting the copyparty container automatically.

#### Finding the Mac's IP for LAN access

```bash
ipconfig getifaddr en0    # WiFi
ipconfig getifaddr en1    # Ethernet (if connected via cable)
```

Or: **System Settings → Network → your active interface → IP address**

Other devices (your Windows 11 PC, phones, tablets) access copyparty at:

```
http://<mac-mini-ip>:3923
```

#### macOS Sequoia — Quick Troubleshooting

| Symptom                                   | Fix                                                                       |
| ----------------------------------------- | ------------------------------------------------------------------------- |
| Drive shows in browser but is empty       | Grant Full Disk Access (Step 2)                                           |
| `permission denied` in logs               | Grant Full Disk Access (Step 2)                                           |
| File transfers are slow                   | Enable VirtioFS in Docker Desktop (Step 3) or switch to OrbStack          |
| Container doesn't start after reboot      | Enable "Start at login" in Docker Desktop / OrbStack settings (Step 7)    |
| Drive name changed after reconnecting USB | Re-run `ls /Volumes/`, update `.env`, restart with `docker compose up -d` |

---

### Platform 3: Ubuntu Server 24.04

#### Step 1 — Install Docker Engine

```bash
curl -fsSL https://get.docker.com | sh
sudo usermod -aG docker $USER   # allow running docker without sudo
newgrp docker                   # apply group without logout
```

#### Step 2 — Identify your drives

```bash
lsblk -f       # shows filesystem labels and UUIDs
df -h          # shows currently mounted paths
```

#### Step 3 — Auto-mount external drives at boot

Create mount points and add entries to `/etc/fstab` so drives mount automatically after every reboot (critical for a headless server):

```bash
sudo mkdir -p /mnt/main /mnt/external1 /mnt/external2
```

Edit `/etc/fstab` (find UUIDs with `lsblk -f`):

```
UUID=xxxx-xxxx  /mnt/external1  exfat  defaults,nofail,uid=1000,gid=1000  0  2
UUID=xxxx-xxxx  /mnt/external2  exfat  defaults,nofail,uid=1000,gid=1000  0  2
```

> The `nofail` flag prevents the server from hanging on boot if a drive is not connected.

Test the fstab entries without rebooting:

```bash
sudo mount -a
df -h   # verify drives appear
```

#### Step 4 — Set up the project

```bash
sudo mkdir -p /opt/copyparty/config
# Copy your project files here, or clone from your git repo:
# git clone <your-repo> /opt/copyparty
```

#### Step 5 — Edit `.env`

```dotenv
CONFIG_PATH=/opt/copyparty/config
MAIN_DRIVE_PATH=/mnt/main
EXT_DRIVE1_PATH=/mnt/external1
EXT_DRIVE2_PATH=/mnt/external2
TZ=America/New_York
CPU_LIMIT=0.50
MEM_LIMIT=256M
```

#### Step 6 — Enable Docker auto-start and run

```bash
sudo systemctl enable docker
docker compose up -d
```

#### Finding the server's IP on Ubuntu

```bash
hostname -I
# or
ip addr show | grep "inet "
```

---

## Network Access from Windows 11 (Mapped Drive)

copyparty exposes WebDAV natively — your Windows 11 PC can mount the Mac's drives as local drive letters in File Explorer with no extra software.

| Method                    | URL format                          |
| ------------------------- | ----------------------------------- |
| Web browser               | `http://<server-ip>:3923`           |
| WebDAV — main drive       | `http://<server-ip>:3923/main`      |
| WebDAV — external drive 1 | `http://<server-ip>:3923/external1` |
| WebDAV — external drive 2 | `http://<server-ip>:3923/external2` |

### Mapping as a Windows Network Drive

1. Open **File Explorer** → right-click **This PC** → **Map network drive**
2. Choose a drive letter (e.g., `Z:`)
3. Folder: `http://<mac-mini-ip>:3923/main`
4. ✅ Check **"Reconnect at sign-in"**
5. Leave credentials blank — click **Finish**

Repeat for each drive using a different letter each time.

> **If WebDAV mapping fails on Windows**, run this in PowerShell as Administrator then try again:
>
> ```powershell
> Set-ItemProperty -Path 'HKLM:\SYSTEM\CurrentControlSet\Services\WebClient\Parameters' -Name BasicAuthLevel -Value 2
> Restart-Service WebClient
> ```

---

## Future Migration to Kubernetes

The `k8s/` directory contains ready-to-use manifests for migrating from `docker compose` to Kubernetes when you move to Ubuntu Server.

### Before deploying

1. **Replace `your-node-name`** in `persistent-volumes.yaml` and `deployment.yaml`:

   ```bash
   kubectl get nodes
   ```

2. **Update host paths** in `persistent-volumes.yaml` to match Ubuntu mount points (e.g., `/mnt/external1`).

3. **Deploy in order:**

   ```bash
   kubectl apply -f k8s/namespace.yaml
   kubectl apply -f k8s/configmap.yaml
   kubectl apply -f k8s/persistent-volumes.yaml
   kubectl apply -f k8s/persistent-volume-claims.yaml
   kubectl apply -f k8s/deployment.yaml
   kubectl apply -f k8s/service.yaml
   ```

4. **Access via NodePort:** `http://<server-ip>:30923`

### Updating config in Kubernetes

```bash
kubectl edit configmap copyparty-config -n copyparty
kubectl rollout restart deployment/copyparty -n copyparty
```

---

## Useful Commands

```bash
# View live logs
docker compose logs -f

# Restart the container
docker compose restart

# Reload config WITHOUT restarting (zero downtime)
docker kill --signal=USR1 copyparty

# Pull latest image and redeploy
docker compose pull && docker compose up -d

# Monitor live resource usage
docker stats copyparty
```

---

## Docker Image Variants

| Image                              | Use case                                       |
| ---------------------------------- | ---------------------------------------------- |
| `ghcr.io/9001/copyparty-ac:latest` | **Default** — Alpine, lightest (~30–70 MB RAM) |
| `ghcr.io/9001/copyparty-im:latest` | Adds ImageMagick for image thumbnails          |
| `ghcr.io/9001/copyparty-iv:latest` | Adds video thumbnail support                   |

To switch variant, update the `image:` line in `docker-compose.yml`.

---

## General Troubleshooting

**Container exits immediately**

```bash
docker compose logs copyparty
```

**Permission denied on drive**

- Linux/macOS: ensure Docker has access to the drive path
- SELinux systems: add `:z` to volume mounts in `docker-compose.yml`

**Drive not showing in browser**

- Verify path in `.env` exists and Docker Desktop has file sharing permissions enabled for that path

**WebDAV not working on Windows**

```powershell
# Run as Administrator
Set-ItemProperty -Path 'HKLM:\SYSTEM\CurrentControlSet\Services\WebClient\Parameters' -Name BasicAuthLevel -Value 2
Restart-Service WebClient
```
