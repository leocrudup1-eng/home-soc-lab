# Custom Rules — EdgeRouter X

## Overview

The EdgeRouter X uses Ubiquiti's syslog format which Wazuh cannot parse with built-in decoders. A custom decoder and custom detection rules were written to provide meaningful alerting on router events.

---

## Custom Decoder

**File:** `local_decoder.xml`  
**Location:** `/var/ossec/etc/decoders/local_decoder.xml`

**Parses:**
EdgeRouter kernel firewall log format:
```
kernel: [WAN_IN-20-A]IN=eth0 OUT=eth2.30 SRC=x.x.x.x DST=y.y.y.y PROTO=TCP
```

Extracts fields:
- `firewall_rule` (e.g. WAN_IN)
- `rule_number`
- `action` (A = accept, D = drop)
- `in_interface`
- `out_interface`
- `srcip`
- `dstip`
- `protocol`

---

## Custom Detection Rules

**File:** `local_rules.xml`  
**Location:** `/var/ossec/etc/rules/local_rules.xml`

| Rule ID | Level | Description | Trigger |
|---|---|---|---|
| 100001 | 5 | EdgeRouter firewall event | Any `UBNT_FW` kernel event |
| 100002 | 10 | SSH failed login | `sshd: Failed password` |
| 100003 | 5 | New DHCP lease | `dhcpd: DHCPACK` |
| 100004 | 12 | SSH successful login | `sshd: Accepted password/publickey` |

**Level guide:** Wazuh alert levels 1-15. Level 12 = high priority (SSH login to router). Level 10 = medium-high (failed SSH attempt). Level 5 = informational.

---

## Testing the Decoder

Use `wazuh-logtest` to verify a log line is parsed correctly:

```bash
/var/ossec/bin/wazuh-logtest
```

Paste a sample EdgeRouter firewall log line:
```
May  5 12:37:38 Router-lab kernel: [WAN_IN-20-A]IN=eth0 OUT=eth2.30 MAC=xx:xx SRC=1.2.3.4 DST=192.168.x.x LEN=52 PROTO=TCP SPT=443 DPT=43502
```

Expected output: decoder `edgerouter-firewall` matched, rule 100001 fired.

---

## Why These Rules Matter

**SSH login to router (Rule 100004 — Level 12):**
The router is the most critical device on the network. Any SSH login should be known and expected. An unexpected login at 3am is a critical incident.

**Failed SSH attempts (Rule 100002 — Level 10):**
Multiple failed SSH attempts = brute force in progress. Combined with Wazuh's built-in frequency rules, repeated failures will generate a higher-level alert automatically.

**New DHCP lease (Rule 100003 — Level 5):**
A new device joining the network is worth knowing about. Unexpected devices could indicate a rogue device, unauthorized access point, or attacker pivot.
