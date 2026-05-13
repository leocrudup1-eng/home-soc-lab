# Component Overview

## Wazuh — SIEM / XDR

**Role:** Central platform for log collection, correlation, alerting, and host-based detection.

**What it does in this lab:**
- Receives logs from all agents (Linux, Windows, network devices)
- Correlates events across sources
- Applies detection rules and generates alerts
- Runs Security Configuration Assessment (SCA) checks against CIS benchmarks
- Performs File Integrity Monitoring (FIM) on critical paths
- Rootcheck scanning on Linux endpoints

**Components deployed:**
- `wazuh-manager` — on Ubuntu Server laptop (central brain)
- `wazuh-agent` — on Librem, Windows Desktop, Windows Server VM
- Remote syslog listener — receives EdgeRouter logs on UDP 514

**Custom work:**
- Written custom decoder for EdgeRouter X firewall log format
- Written custom detection rules for router events (SSH, firewall, DHCP)

---

## Suricata — Network IDS

**Role:** Passive network intrusion detection system watching all traffic via SPAN port.

**What it does in this lab:**
- Inspects all traffic mirrored from the managed switch
- Applies Emerging Threats (ET) Open ruleset
- Generates alerts in `eve.json` and `fast.log`
- Forwards alerts to Wazuh for correlation

**Configuration highlights:**
- AF_PACKET capture mode (zero-copy, efficient)
- Single interface, promiscuous mode
- ET Open ruleset, tuned with suppression rules to eliminate lab noise
- Suppressed: stream errors, known admin tools, Pi-hole DNS patterns

**Deployed on:** Ubuntu Server laptop (receives mirror port traffic)

---

## Pi-hole — DNS Filtering

**Role:** Network-level DNS sinkhole blocking known malicious/ad domains for all lab traffic.

**What it does in this lab:**
- Acts as upstream DNS resolver for all devices
- Blocks domains on known blocklists
- Logs all DNS queries (visibility into what devices are resolving)

**Deployed on:** Raspberry Pi 3

---

## EdgeRouter X — Router / Firewall

**Role:** Network perimeter device, provides WAN routing and firewall policy.

**What it does in this lab:**
- Routes all traffic between LAN and internet
- Enforces firewall rules (WAN_IN, WAN_LOCAL policies)
- Forwards syslog to Wazuh Manager for centralized log ingestion
- Events captured: firewall blocks, SSH logins, DHCP leases, config changes

**Integration:** Syslog forwarded to Wazuh Manager via UDP 514

---

## Ugreen CM933 — Managed Switch

**Role:** Layer 2 switching with port mirroring capability.

**What it does in this lab:**
- Connects all lab devices on a single LAN
- Mirrors all port traffic to the Suricata interface (passive IDS tap)

**Key feature used:** Port mirroring (SPAN) — allows Suricata to see all traffic passively

---

## Sysmon — Windows Telemetry

**Role:** Provides rich process, network, file, and registry telemetry on Windows endpoints.

**What it does in this lab:**
- Logs all process creation events with full command lines
- Logs network connections with process context
- Logs registry modifications
- Logs file creation/modification
- All events forwarded to Wazuh via Windows Event Log

**Deployed on:** Windows Desktop (DESKTOP-A904EEG)

**Value:** Sysmon turns basic Windows event logs into SOC-grade telemetry. Without it, Windows endpoints generate minimal useful security data.

---

## auditd — Linux Syscall Auditing

**Role:** Kernel-level auditing of all syscalls on the Librem Linux laptop.

**What it does in this lab:**
- Captures privileged command execution
- Logs file access on sensitive paths
- Records authentication events
- All logs forwarded to Wazuh agent

**Deployed on:** Librem Linux laptop

---

## How It All Fits Together

```
Detection Layer         Collection Layer        Analysis Layer
─────────────           ─────────────────       ──────────────
Suricata (network)  →   Wazuh agents        →   Wazuh Manager
Sysmon (Windows)    →   Wazuh agents        →   Dashboard
auditd (Linux)      →   Wazuh agents        →   Custom rules
EdgeRouter (net)    →   Syslog UDP 514      →   Custom decoder
Pi-hole (DNS)       →   (local logs)        →   (future integration)
```
