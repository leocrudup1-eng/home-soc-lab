# Pi-hole Setup Guide

## Environment
- **Host:** Raspberry Pi 3
- **OS:** Raspberry Pi OS (Lite recommended)
- **Role:** DNS-level threat blocking and query logging for all lab traffic

---

## 1. Install Pi-hole

```bash
curl -sSL https://install.pi-hole.net | bash
```

Follow the interactive installer. Key choices:
- **Interface:** `eth0` (wired recommended over WiFi for reliability)
- **Upstream DNS:** Choose a provider (e.g. Cloudflare `1.1.1.1`, Google `8.8.8.8`)
- **Install web admin:** Yes
- **Enable logging:** Yes

---

## 2. Set a Static IP on the Pi

Edit `/etc/dhcpcd.conf`:

```bash
interface eth0
static ip_address=192.168.x.x/24     # choose a fixed IP
static routers=192.168.x.1           # your router IP
static domain_name_servers=127.0.0.1  # point to itself
```

Restart networking:
```bash
sudo systemctl restart dhcpcd
```

---

## 3. Point All Devices to Pi-hole

On the EdgeRouter, set the DHCP server to hand out the Pi's IP as the DNS server:

```bash
configure
set service dhcp-server shared-network-name LAN subnet 192.168.x.0/24 dns-server 192.168.x.x
commit
save
```

Replace `192.168.x.x` with the Pi's static IP.

---

## 4. Access the Admin Dashboard

```
http://192.168.x.x/admin
```

Default password is set during installation.

---

## 5. Verify It's Working

From any device on the network:
```bash
nslookup google.com 192.168.x.x
```

Should resolve. Then try a known ad domain — it should return `0.0.0.0`.

---

## Maintenance

Update Pi-hole:
```bash
pihole -up
```

Update blocklists:
```bash
pihole -g
```

Check status:
```bash
pihole status
```
