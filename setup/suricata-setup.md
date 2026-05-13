# Suricata Setup Guide

## Environment
- **Host:** Ubuntu Server 24.04
- **Capture method:** SPAN port (passive mirror from CM933 switch)
- **Suricata version:** 7.x

---

## 1. Install Suricata

```bash
apt update && apt install -y suricata
```

---

## 2. Configure SPAN Port on Ugreen CM933

Log into the CM933 web interface (find its IP via `arp -a` or your router's DHCP list).

Navigate to **Monitoring → Port Mirroring** and configure:

| Setting | Value |
|---|---|
| Source ports | All active ports |
| Direction | Ingress + Egress |
| Destination port | Port connected to Ubuntu Server |
| Status | Enabled |

---

## 3. Set Interface to Promiscuous Mode

```bash
# Temporary (survives until reboot)
ip link set eth0 promisc on

# Permanent — add to /etc/rc.local before exit 0
ip link set eth0 promisc on
```

Replace `eth0` with the actual interface name connected to the mirror port (`ip a` to list interfaces).

---

## 4. Configure Suricata

Edit `/etc/suricata/suricata.yaml`:

**Set HOME_NET:**
```yaml
vars:
  address-groups:
    HOME_NET: "[192.168.x.0/24]"   # replace with your subnet
    EXTERNAL_NET: "!$HOME_NET"
```

**Set capture interface:**
```yaml
af-packet:
  - interface: eth0    # replace with your interface
    threads: auto
    cluster-type: cluster_flow
    buffer-size: 32768
    use-mmap: yes
```

---

## 5. Update Rules

```bash
suricata-update
```

To disable noisy categories not relevant to a home lab, create `/etc/suricata/disable.conf`:

```
group:emerging-games
group:emerging-p2p
group:emerging-voip
group:emerging-spam
group:emerging-inappropriate
```

Apply:
```bash
suricata-update --disable-conf /etc/suricata/disable.conf
```

---

## 6. Test Configuration

```bash
suricata -T -c /etc/suricata/suricata.yaml -v
```

Should end with: `Configuration provided was successfully loaded.`

---

## 7. Start Suricata

```bash
systemctl enable suricata
systemctl start suricata
systemctl status suricata
```

---

## 8. Verify Alerts Are Generating

Watch live alerts:
```bash
tail -f /var/log/suricata/fast.log
```

JSON format (more detail):
```bash
tail -f /var/log/suricata/eve.json | python3 -m json.tool
```

Filter for alerts only:
```bash
tail -f /var/log/suricata/eve.json | jq 'select(.event_type=="alert")'
```

Test detection by running an nmap scan from another device:
```bash
nmap -sS 192.168.x.x
```

You should see alerts fire in `fast.log` immediately.

---

## 9. Configure Log Rotation

Prevent `eve.json` from filling the disk:

```bash
nano /etc/logrotate.d/suricata
```

```
/var/log/suricata/*.log /var/log/suricata/*.json {
    daily
    rotate 7
    compress
    missingok
    sharedscripts
    postrotate
        systemctl kill -s HUP suricata
    endscript
}
```

---

## 10. Integrate with Wazuh

The Wazuh agent on the Ubuntu Server reads Suricata's `eve.json` and forwards alerts to the Wazuh Manager.

In the Wazuh agent config (`/var/ossec/etc/ossec.conf`), verify this localfile block exists:

```xml
<localfile>
  <log_format>json</log_format>
  <location>/var/log/suricata/eve.json</location>
</localfile>
```

Restart agent:
```bash
systemctl restart wazuh-agent
```

Suricata alerts should now appear in the Wazuh dashboard under **Security Events**.

---

## Suppression Rules

See [`configurations/suricata-suppress.md`](../configurations/suricata-suppress.md) for suppression rules added to reduce lab noise, with rationale for each.

---

## Troubleshooting

**No alerts firing:**
```bash
# Confirm interface is seeing traffic
tcpdump -i eth0 -c 10

# Confirm promiscuous mode is on
ip link show eth0 | grep PROMISC
```

**High packet drop rate:**
```bash
grep kernel_drops /var/log/suricata/stats.log
```
If drops are high, reduce monitored ports on the switch mirror.

**Check Suricata stats:**
```bash
tail -f /var/log/suricata/stats.log
```
