# Network Architecture

## Topology

```
                        ┌─────────────────────┐
                        │     INTERNET         │
                        └──────────┬──────────┘
                                   │
                        ┌──────────▼──────────┐
                        │   EdgeRouter X       │
                        │   (Router/Firewall)  │
                        │   Syslog → UDP 514   │
                        └──────────┬──────────┘
                                   │
                        ┌──────────▼──────────┐
                        │  Ugreen CM933        │
                        │  Managed Switch      │
                        │  Port mirroring ON   │
                        └──┬──┬──┬──┬─────────┘
                           │  │  │  │
              ┌────────────┘  │  │  └──────────────────┐
              │               │  │                      │
    ┌─────────▼──────┐  ┌────▼──▼────────┐  ┌─────────▼────────┐
    │ Raspberry Pi 3  │  │ Ubuntu Server   │  │  Librem Laptop   │
    │ Pi-hole         │  │ Laptop          │  │  Wazuh Agent     │
    │ DNS filtering   │  │                 │  │  auditd          │
    └────────────────┘  │ Wazuh Manager   │  │                  │
                        │ Suricata IDS    │  │  ┌─────────────┐ │
                        │ ← Mirror port  │  │  │Windows Svr VM│ │
                        └────────────────┘  │  │Wazuh Agent  │ │
                                            │  └─────────────┘ │
                                            └──────────────────┘
                                 ┌───────────────────┐
                                 │ Windows Desktop    │
                                 │ DESKTOP-A904EEG    │
                                 │ Wazuh Agent        │
                                 │ Sysmon             │
                                 └───────────────────┘
```

---

## Data Flow

### Network Traffic (IDS)
```
All switch traffic
    → Port mirror (CM933)
    → Suricata on Ubuntu Server
    → eve.json / fast.log
    → Wazuh agent forwards alerts
    → Wazuh Manager (indexed + correlated)
```

### Router Logs
```
EdgeRouter X
    → Syslog UDP 514
    → Wazuh Manager remote listener
    → Custom decoder (local_decoder.xml)
    → Custom rules (local_rules.xml)
    → Wazuh alerts dashboard
```

### Host Telemetry — Linux (Librem)
```
Librem Linux
    → auditd (syscall events)
    → Wazuh Agent
    → Wazuh Manager
```

### Host Telemetry — Windows (Desktop + Server VM)
```
Windows endpoints
    → Sysmon (process, network, registry events)
    → Windows Event Log
    → Wazuh Agent reads Event Log
    → Wazuh Manager
```

### DNS Filtering
```
All lab DNS queries
    → Pi-hole (Raspberry Pi)
    → Upstream resolver
    → Blocked domains logged
```

---

## Network Segments

| Segment | Subnet | Devices |
|---|---|---|
| Main LAN | 192.168.x.0/24 | All lab devices |

> Note: Future plan includes VLAN segmentation and a dedicated honeypot subnet.

---

## Port Mirroring Configuration

The Ugreen CM933 is configured to mirror all switch port traffic to the port connected to the Ubuntu Server laptop. This gives Suricata passive visibility into all east-west and north-south traffic without being inline.

- **Mirror source:** All active ports
- **Mirror destination:** Ubuntu Server NIC (Suricata interface)
- **Suricata mode:** Promiscuous, AF_PACKET capture

---

## Key Design Decisions

**Why SPAN port mirroring instead of inline IDS?**
Passive monitoring means Suricata failure doesn't affect network connectivity. Inline IDS creates a single point of failure — not appropriate for a home network.

**Why Wazuh instead of Splunk/ELK?**
Wazuh is free, actively maintained, purpose-built for security monitoring, and includes HIDS, FIM, SCA, and SIEM in one platform. ELK requires significantly more resources for the same visibility.

**Why Pi-hole on a dedicated Pi?**
Isolating DNS filtering to its own low-power device keeps it running even when other lab machines are down. DNS filtering should be always-on.
