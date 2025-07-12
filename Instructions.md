# 🧲 Torrenting Proxmox LXC (VPN for p2p traffic and tailscale for Remote Access)

Run **qBittorrent, Radarr, Sonarr, Prowlarr, Gluetun,** and **Portainer** inside an unprivileged Ubuntu-based LXC (including docker and tailscale). The container has read‑write access to `/mnt/shared_data` on the host.

* * *

## 🔧 Host Setup (Proxmox)

1.  **Create a shared group for `/mnt/shared_data`**

```bash
sudo groupadd -g 101000 shared_data
sudo chown -R root:shared_data /mnt/shared_data
sudo chmod 2770 /mnt/shared_data
```

2.  **Allow `/dev/net/tun` and bind mounts for the unprivileged LXC**

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
net0: name=eth0,bridge=vmbr0,gw=192.168.8.1,ip=192.168.8.199/24,firewall=1,hwaddr=BC:24:11:1A:D8:08

# shared storage + VPN tunnel
lxc.mount.entry = /mnt/shared_data shared_data none bind,create=dir
lxc.mount.entry = /dev/net/tun   dev/net/tun none bind,create=file
lxc.mount.entry: /dev/net dev/net none bind,create=file
lxc.cgroup2.devices.allow = c 10:200 rwm
lxc.cgroup.devices.allow  = a
```

*If you prefer the GUI, you can use an `mp0` bind instead of the manual `lxc.mount.entry`.*

* * *

## 🐧 Inside the Container

### 3\. Match UID/GID mapping

```bash
groupadd -g 1000 shared_data
usermod -aG shared_data $USER    # $USER is usually “root” inside an unprivileged LXC
```

UID 1000 / GID 1000 in the container map to UID 101000 / GID 101000 on the host, giving the stack write access to `/mnt/shared_data`.

### 4\. Install Docker Engine **and** the Compose plugin

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

### 5\. Install Portainer for a web‑UI

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

Open **`https://<container-IP>:9443`** (or `http://…:9000`) to finish the Portainer wizard.

### 6\. Install and connect Tailscale

```bash
curl -fsSL https://tailscale.com/install.sh | sh
systemctl enable --now tailscaled
tailscale up --hostname=torrenting --ssh
```

* * *

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
      - TZ=Australia/Melbourne
      - VPN_SERVICE_PROVIDER=surfshark
      - VPN_TYPE=wireguard
      - WIREGUARD_PRIVATE_KEY=<redacted>
      - WIREGUARD_ADDRESSES=10.14.0.2/16
      - SERVER_HOSTNAMES=au-mel.prod.surfshark.com
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
      - TZ=Australia/Melbourne
      - WEBUI_PORT=5080
    volumes:
      - qbittorrent-config:/config
      - /shared_data:/shared_data:rw
    depends_on:
      - gluetun
    restart: unless-stopped

  radarr:
    image: lscr.io/linuxserver/radarr:latest
    container_name: radarr
    environment:
      - PUID=1000
      - PGID=1000
      - TZ=Australia/Melbourne
    volumes:
      - radarr-config:/config
      - /shared_data:/shared_data:rw
    ports:
      - 7878:7878
    restart: unless-stopped

  sonarr:
    image: lscr.io/linuxserver/sonarr:latest
    container_name: sonarr
    environment:
      - PUID=1000
      - PGID=1000
      - TZ=Australia/Melbourne
    volumes:
      - sonarr-config:/config
      - /shared_data:/shared_data:rw
    ports:
      - 8989:8989
    restart: unless-stopped

  prowlarr:
    image: lscr.io/linuxserver/prowlarr:latest
    container_name: prowlarr
    environment:
      - PUID=1000
      - PGID=1000
      - TZ=Australia/Melbourne
    volumes:
      - prowlarr-config:/config
      - /shared_data:/shared_data:rw
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

&nbsp;

## ✅ Summary

| Component / Path | Host UID/GID ↔ LXC UID/GID | Notes |
| --- | --- | --- |
| `/mnt/shared_data` | 101000 ↔ 1000 | Group `shared_data`; perms `2770` (setgid). |
| **Docker Engine + Compose** | –   | Installed from Docker repo; `docker compose` CLI available. |
| **Portainer** | –   | Web UI (ports 9000 / 9443); data in `portainer_data`. |
| **Gluetun VPN** | needs `/dev/net/tun` & `CAP_NET_ADMIN` | Provides network namespace for the stack. |
| **Torrenting Apps** | run as UID/GID 1000 | qBittorrent, Radarr, Sonarr, Prowlarr all bind‑mount `/shared_data`. |
| **Tailscale** | –   | `tailscale up --ssh` for Tailnet & secure SSH. |
| **LXC Type** | unprivileged | `nesting=1`; custom cgroup & mount entries for TUN and shared storage. |
