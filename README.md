# Home SOC Lab

> A fully functional Security Operations Center built from consumer hardware, running enterprise-grade open source tools. Built to develop real blue team skills — not just certifications.

---

## Overview

This lab simulates a real SOC environment with network-level intrusion detection, host-based monitoring across multiple endpoints, centralized log management, custom detection rules, and documented alert triage methodology.

Everything here was built, broken, debugged, and tuned hands-on.

---

## Architecture

```
Internet
    |
[EdgeRouter X] ──syslog──────────────────────────┐
    |                                             |
[Ugreen CM933 Managed Switch]             [Ubuntu Server Laptop]
    |── Port Mirror ──────────────────────→  Suricata (IDS)
    |                                        Wazuh Manager (SIEM)
    |── [Raspberry Pi 3]                          ↑
    |       └── Pi-hole (DNS filtering)           |
    |                                             |
    |── [Librem Linux Laptop] ──Wazuh agent───────┤
    |       └── auditd                            |
    |       └── [Windows Server VM] ──agent───────┤
    |                                             |
    └── [Windows Desktop] ──Wazuh agent───────────┘
            └── Sysmon
```

---

## Lab Inventory

| Device | OS | Role | Monitoring |
|---|---|---|---|
| Ubiquiti EdgeRouter X | EdgeOS | Router / Firewall | Syslog → Wazuh |
| Ugreen CM933 | — | Managed Switch | Port mirroring to Suricata |
| Raspberry Pi 3 | Raspberry Pi OS | DNS filtering | Pi-hole |
| Ubuntu Server Laptop | Ubuntu Server 24.04 | SIEM + IDS | Wazuh Manager + Suricata |
| Librem Linux Laptop | PureOS | Endpoint + VM Host | Wazuh Agent + auditd |
| Windows Server VM | Windows Server | Domain / Endpoint | Wazuh Agent |
| Windows Desktop | Windows 10 | Endpoint | Wazuh Agent + Sysmon |

---

## Tools Deployed

| Tool | Version | Role |
|---|---|---|
| [Wazuh](https://wazuh.com) | 4.x | SIEM / XDR / Host monitoring |
| [Suricata](https://suricata.io) | 7.x | Network IDS (SPAN port) |
| [Pi-hole](https://pi-hole.net) | Latest | DNS-level threat blocking |
| [Sysmon](https://learn.microsoft.com/en-us/sysinternals/downloads/sysmon) | Latest | Windows process/event telemetry |
| auditd | — | Linux syscall auditing |
| [ServiceNow](https://developer.servicenow.com) | Developer instance | Incident ticketing (ITSM) |

---

## What I Built

- **Passive network IDS** via SPAN port mirroring on the managed switch — Suricata sees all traffic without being inline
- **Multi-source SIEM** ingesting logs from 5 agents across Linux, Windows, and network devices
- **Custom Wazuh decoder** for EdgeRouter X firewall log format (Ubiquiti syslog)
- **Custom Wazuh detection rules** for EdgeRouter events (firewall blocks, SSH login, DHCP)
- **DNS-level filtering** via Pi-hole on all lab traffic
- **Alert triage methodology** developed through real hands-on investigation sessions
- **Suricata rule tuning** — suppression rules calibrated to eliminate false positives while preserving real detection capability
- **ServiceNow integration** — Wazuh automatically creates incident tickets in ServiceNow for all rule level 12+ alerts via a custom Python integration script and REST API

---

## Repository Structure

```
home-soc-lab/
├── README.md
├── architecture/
│   ├── network-diagram.md          ← topology and data flow
│   └── component-overview.md       ← what each tool does and why
├── setup/
│   ├── wazuh-setup.md              ← Wazuh manager + agent installation
│   ├── suricata-setup.md           ← Suricata + SPAN port setup
│   ├── pihole-setup.md             ← Pi-hole on Raspberry Pi
│   ├── edgerouter-syslog-setup.md  ← syslog forwarding to Wazuh
│   └── servicenow-integration.md  ← ServiceNow ITSM integration
├── configurations/
│   ├── local_rules.xml             ← custom Wazuh detection rules
│   ├── local_decoder.xml           ← custom EdgeRouter decoder
│   └── suricata-suppress.md        ← suppression rules + rationale
├── custom-rules/
│   ├── edgerouter-rules.md         ← rules written for EdgeRouter events
│   └── suppression-rationale.md    ← why each suppression was added
├── triage-reports/
│   └── 2026-05-05-session.md       ← full triage session with verdicts
└── lessons-learned/
    └── notes.md                    ← things learned the hard way
```

---

## Skills Demonstrated

- Network traffic analysis and IDS tuning
- SIEM deployment and multi-source log ingestion
- Custom decoder and detection rule writing (XML)
- Alert triage and false positive identification
- Host-based monitoring (Linux auditd, Windows Sysmon)
- Network segmentation concepts
- Blue team investigation methodology
- Technical documentation

---

## Certifications

- CompTIA Security+
- CompTIA Network+
- Cisco CCNA
- CompTIA CySA+ *(in progress)*

---

## Topics

`soc-home-lab` `wazuh` `suricata` `ids` `siem` `blue-team` `network-security` `cybersecurity` `incident-response` `log-analysis` `alert-triage` `raspberry-pi` `edgerouter` `sysmon` `auditd`
