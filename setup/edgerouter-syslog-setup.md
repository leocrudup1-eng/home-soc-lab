# EdgeRouter X — Syslog Forwarding to Wazuh

## Overview

The EdgeRouter X forwards all syslog events to the Wazuh Manager over UDP 514. A custom decoder parses the EdgeRouter firewall log format, and custom rules generate alerts for relevant events.

---

## 1. Enable Firewall Logging on EdgeRouter

SSH into the router:
```bash
ssh admin@192.168.x.1
```

Enable logging on WAN firewall rules:
```bash
configure

set firewall name WAN_IN rule 1 log enable
set firewall name WAN_LOCAL rule 1 log enable

commit
save
```

---

## 2. Configure Syslog Forwarding

```bash
configure

set system syslog host 192.168.x.x facility all level info   # replace with Wazuh manager IP
set system syslog host 192.168.x.x port 514
set system syslog host 192.168.x.x protocol udp

commit
save
exit
```

Verify:
```bash
show system syslog
```

---

## 3. Verify Logs Are Arriving at Wazuh Manager

On the Wazuh Manager machine:
```bash
tcpdump -i any port 514
```

Then trigger a router event (SSH login, new DHCP device). You should see syslog packets arriving.

---

## 4. Custom Decoder

The EdgeRouter uses Ubiquiti's syslog format which Wazuh cannot parse with built-in decoders. A custom decoder was written for the `kernel: [WAN_...]` firewall log format.

See: [`configurations/local_decoder.xml`](../configurations/local_decoder.xml)

---

## 5. Custom Detection Rules

Custom rules alert on:
- Firewall block events (`[WAN_IN-xxx-D]`)
- Successful SSH logins to the router
- Failed SSH login attempts (potential brute force)
- New DHCP leases (new device joined network)

See: [`configurations/local_rules.xml`](../configurations/local_rules.xml)

---

## Events Captured

| Event | Source | Wazuh Rule |
|---|---|---|
| WAN firewall block | kernel syslog | 100001 |
| SSH login success | sshd | 100004 |
| SSH login failure | sshd | 100002 |
| New DHCP lease | dhcpd | 100003 |

---

## Troubleshooting

**Logs arriving but not matching rules:**
```bash
# Use wazuh-logtest to manually test a log line
/var/ossec/bin/wazuh-logtest
```
Paste a sample EdgeRouter log line — it will show which decoder and rule matched (or why it didn't).

**Check if Wazuh is receiving from the router IP:**
```bash
grep "192.168.x.1" /var/ossec/logs/archives/archives.log | tail -20
```
