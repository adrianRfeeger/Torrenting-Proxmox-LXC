# 🧲 Torrenting Stack in a Proxmox LXC  
(VPN-protected P2P traffic + Tailscale remote access)

These instructions help you spin up **Gluetun (VPN), qBittorrent, Radarr, Sonarr, Prowlarr** and **Portainer** in an *unprivileged* Ubuntu LXC. Knowledge of Proxmox, Docker, Portainer, *arr tools (Prolarr, Sonarr, Radarr etc) and Tailscale is required.

Replace the **bold‐caps placeholders** with values that suit **your** LAN, storage path and VPN provider.

## 🎯 What This Does

This setup creates a **completely isolated torrenting environment** that:

- **🔒 Routes ALL P2P traffic through a VPN** - Your ISP never sees torrent traffic
- **🏠 Keeps management traffic on your LAN** - Web UIs accessible locally without VPN
- **🌐 Provides secure remote access** - Tailscale lets you manage from anywhere
- **📁 Shares storage with your host** - Downloads go directly to your NAS/server storage
- **🐳 Runs everything in Docker** - Easy updates, backups, and management

### How It Works

1. **Gluetun** creates a VPN tunnel and acts as a network gateway
2. **qBittorrent** runs entirely through Gluetun's network (VPN-only)
3. **Radarr/Sonarr/Prowlarr** connect to qBittorrent and manage downloads
4. **All apps** share the same storage directory for seamless file access
5. **Tailscale** provides encrypted remote access without exposing ports
6. **Portainer** gives you a web interface to manage all containers

The genius is that **only P2P traffic goes through the VPN** - everything else uses your normal internet connection for speed and reliability.

---

## 📋 Variable cheat-sheet

| Placeholder | Example | Description |
|-------------|---------|-------------|
| **`<VMID>`** | `105` | Proxmox CT ID you create |
| **`<LAN_GW>`** | `192.168.8.1` | Your router's gateway IP |
| **`<CT_IP>`** | `192.168.8.199` | Static IP you assign to the container |
| **`<SUBNET>`** | `24` | CIDR mask (e.g. 24 = 255.255.255.0) |
| **`<MAC_ADDR>`** | `BC:24:11:1A:D8:08` | MAC address for the container (optional) |
| **`<SHARE_PATH>`** | `/mnt/shared_data` | Host directory you want the stack to use |
| **`<VPN_PRIV_KEY>`** | `<redacted>` | Your WireGuard private key |
| **`<VPN_ADDRESSES>`** | `10.14.0.2/16` | Your VPN client IP addresses |
| **`<VPN_SERVER>`** | `au-mel.prod.surfshark.com` | VPN exit server hostname |
| **`<TZ>`** | `Australia/Melbourne` | Time-zone for logs & cron |

---

## 🔧 Host Setup (Proxmox)

### 1. Create a shared group for `<SHARE_PATH>`

```bash
sudo groupadd -g 101000 shared_data
sudo mkdir -p <SHARE_PATH>
sudo chown -R root:shared_data <SHARE_PATH>
sudo chmod 2770 <SHARE_PATH>
```

### 2. Allow `/dev/net/tun` and bind mounts for the unprivileged LXC

Edit **`/etc/pve/lxc/<VMID>.conf`**:

```conf
arch: amd64
cores: 2
memory: 4096
swap: 512
hostname: torrenting
ostype: ubuntu
rootfs: local-zfs:subvol-<VMID>-disk-0,size=32G
unprivileged: 1
onboot: 1
features: nesting=1
nameserver: 1.1.1.1
net0: name=eth0,bridge=vmbr0,gw=<LAN_GW>,ip=<CT_IP>/<SUBNET>,firewall=1,hwaddr=<MAC_ADDR>

# shared storage + VPN tunnel
lxc.mount.entry = <SHARE_PATH> shared_data none bind,create=dir
lxc.mount.entry = /dev/net/tun   dev/net/tun none bind,create=file
lxc.mount.entry: /dev/net dev/net none bind,create=file
lxc.cgroup2.devices.allow = c 10:200 rwm
lxc.cgroup.devices.allow  = a
```
---

## 🐧 Inside the Container

### 3. Match UID/GID mapping

```bash
groupadd -g 1000 shared_data
usermod -aG shared_data $USER    # $USER is usually "root" inside an unprivileged LXC
```

UID 1000 / GID 1000 in the container map to UID 101000 / GID 101000 on the host, giving the stack write access to `<SHARE_PATH>`.

### 4. Install Docker Engine **and** the Compose plugin

```bash
apt update && apt install -y ca-certificates curl gnupg lsb-release

# Docker repo
install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg |
  gpg --dearmor -o /etc/apt/keyrings/docker.gpg

echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] \
  https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" \
  | tee /etc/apt/sources.list.d/docker.list >/dev/null

apt update
apt install -y docker-ce docker-ce-cli containerd.io \
               docker-buildx-plugin docker-compose-plugin
systemctl enable --now docker
```

### 5. Install Portainer for a web‑UI

```bash
docker volume create portainer_data
docker run -d \
  --name portainer \
  --restart=unless-stopped \
  -p 9000:9000 -p 9443:9443 \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v portainer_data:/data \
  portainer/portainer-ce:latest
```

Open **`https://<CT_IP>:9443`** (or `http://<CT_IP>:9000`) to finish the Portainer wizard.

### 6. Install and connect Tailscale

```bash
curl -fsSL https://tailscale.com/install.sh | sh
systemctl enable --now tailscaled
tailscale up --hostname=torrenting --ssh
```

---

## 🐳 Portainer Stack Compose file

```yaml
name: torrenting-stack
services:
  gluetun:
    image: qmcgaw/gluetun:latest
    container_name: gluetun
    cap_add:
      - NET_ADMIN
    devices:
      - /dev/net/tun
    environment: # Use your own VPN settings
      - TZ=<TZ>
      - VPN_SERVICE_PROVIDER=surfshark
      - VPN_TYPE=wireguard
      - WIREGUARD_PRIVATE_KEY=<VPN_PRIV_KEY>
      - WIREGUARD_ADDRESSES=<VPN_ADDRESSES>
      - SERVER_HOSTNAMES=<VPN_SERVER>
      - DNS=162.252.172.57,149.154.159.92
    volumes:
      - gluetun-config:/gluetun
    ports:
      - 5080:5080   # qBittorrent Web UI (tunnelled)
    restart: unless-stopped

  qbittorrent:
    image: lscr.io/linuxserver/qbittorrent:latest
    container_name: qbittorrent
    network_mode: "service:gluetun"
    environment:
      - PUID=1000
      - PGID=1000
      - TZ=<TZ>
      - WEBUI_PORT=5080
    volumes:
      - qbittorrent-config:/config
      - <SHARE_PATH>:/shared_data:rw
    depends_on:
      - gluetun
    restart: unless-stopped

  radarr:
    image: lscr.io/linuxserver/radarr:latest
    container_name: radarr
    environment:
      - PUID=1000
      - PGID=1000
      - TZ=<TZ>
    volumes:
      - radarr-config:/config
      - <SHARE_PATH>:/shared_data:rw
    ports:
      - 7878:7878
    restart: unless-stopped

  sonarr:
    image: lscr.io/linuxserver/sonarr:latest
    container_name: sonarr
    environment:
      - PUID=1000
      - PGID=1000
      - TZ=<TZ>
    volumes:
      - sonarr-config:/config
      - <SHARE_PATH>:/shared_data:rw
    ports:
      - 8989:8989
    restart: unless-stopped

  prowlarr:
    image: lscr.io/linuxserver/prowlarr:latest
    container_name: prowlarr
    environment:
      - PUID=1000
      - PGID=1000
      - TZ=<TZ>
    volumes:
      - prowlarr-config:/config
      - <SHARE_PATH>:/shared_data:rw
    ports:
      - 9696:9696
    restart: unless-stopped

volumes:
  radarr-config:
  sonarr-config:
  prowlarr-config:
  qbittorrent-config:
  gluetun-config:
```

---

## 🎮 Post-Setup Configuration

### Configure qBittorrent for VPN-only traffic

Once the stack is running, you **must** configure qBittorrent to use only the VPN interface:

1. **Access qBittorrent** at `http://<CT_IP>:5080`
2. **Login** with default credentials: `admin` / `adminadmin`
3. **Go to Settings** → **Advanced** → **Network Interface**
4. **Set Network Interface** to `tun0` (this forces all P2P traffic through the VPN)
5. **Save** and restart qBittorrent

⚠️ **CRITICAL**: Without setting the interface to `tun0`, torrents may leak through your regular internet connection!

### Access your services

- **qBittorrent**: `http://<CT_IP>:5080` (VPN-protected torrenting)
- **Radarr**: `http://<CT_IP>:7878` (Movie management)
- **Sonarr**: `http://<CT_IP>:8989` (TV show management)  
- **Prowlarr**: `http://<CT_IP>:9696` (Indexer management)
- **Portainer**: `https://<CT_IP>:9443` (Container management)

---

## ✅ Summary

| Component / Path | Host UID/GID ↔ LXC UID/GID | Notes |
| --- | --- | --- |
| `<SHARE_PATH>` | 101000 ↔ 1000 | Group `shared_data`; perms `2770` (setgid). |
| **Docker Engine + Compose** | –   | Installed from Docker repo; `docker compose` CLI available. |
| **Portainer** | –   | Web UI (ports 9000 / 9443); data in `portainer_data`. |
| **Gluetun VPN** | needs `/dev/net/tun` & `CAP_NET_ADMIN` | Provides network namespace for the stack. |
| **Torrenting Apps** | run as UID/GID 1000 | qBittorrent, Radarr, Sonarr, Prowlarr all bind‑mount `/shared_data`. |
| **Tailscale** | –   | `tailscale up --ssh` for Tailnet & secure SSH. |
| **LXC Type** | unprivileged | `nesting=1`; custom cgroup & mount entries for TUN and shared storage. |
