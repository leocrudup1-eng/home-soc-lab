# Wazuh Setup Guide

## Environment
- **Manager host:** Ubuntu Server 24.04
- **Wazuh version:** 4.x

---

## 1. Install Wazuh Manager

```bash
# Add Wazuh repository
curl -s https://packages.wazuh.com/key/GPG-KEY-WAZUH | gpg --no-default-keyring \
  --keyring gnupg-ring:/usr/share/keyrings/wazuh.gpg --import
chmod 644 /usr/share/keyrings/wazuh.gpg

echo "deb [signed-by=/usr/share/keyrings/wazuh.gpg] https://packages.wazuh.com/4.x/apt/ \
  stable main" | tee /etc/apt/sources.list.d/wazuh.list

apt update
apt install wazuh-manager

# Enable and start
systemctl daemon-reload
systemctl enable wazuh-manager
systemctl start wazuh-manager
systemctl status wazuh-manager
```

---

## 2. Install Wazuh Dashboard (Web UI)

Follow the official Wazuh documentation for the full stack install (Wazuh indexer + dashboard):
https://documentation.wazuh.com/current/installation-guide/

---

## 3. Configure Remote Syslog Listener (for EdgeRouter)

Edit `/var/ossec/etc/ossec.conf` and add inside `<ossec_config>`:

```xml
<remote>
  <connection>syslog</connection>
  <port>514</port>
  <protocol>udp</protocol>
  <allowed-ips>192.168.x.1</allowed-ips>  <!-- replace with EdgeRouter IP -->
</remote>
```

Open firewall:
```bash
ufw allow 514/udp
ufw allow 514/tcp
```

Restart:
```bash
systemctl restart wazuh-manager
```

Verify listening:
```bash
ss -tulnp | grep 514
```

---

## 4. Deploy Wazuh Agent — Linux (Librem)

On the agent machine:
```bash
# Add repo (same as manager)
curl -s https://packages.wazuh.com/key/GPG-KEY-WAZUH | gpg --no-default-keyring \
  --keyring gnupg-ring:/usr/share/keyrings/wazuh.gpg --import
chmod 644 /usr/share/keyrings/wazuh.gpg

echo "deb [signed-by=/usr/share/keyrings/wazuh.gpg] https://packages.wazuh.com/4.x/apt/ \
  stable main" | tee /etc/apt/sources.list.d/wazuh.list

apt update
WAZUH_MANAGER='192.168.x.x' apt install wazuh-agent  # replace with manager IP

systemctl daemon-reload
systemctl enable wazuh-agent
systemctl start wazuh-agent
```

---

## 5. Deploy Wazuh Agent — Windows

Download the MSI installer from the Wazuh dashboard or:
https://documentation.wazuh.com/current/installation-guide/wazuh-agent/wazuh-agent-package-windows.html

During install, set the manager IP to your Ubuntu Server laptop's IP.

Or silent install via PowerShell:
```powershell
Invoke-WebRequest -Uri https://packages.wazuh.com/4.x/windows/wazuh-agent-4.x.x-1.msi `
  -OutFile wazuh-agent.msi

msiexec.exe /i wazuh-agent.msi /q WAZUH_MANAGER='192.168.x.x'

NET START WazuhSvc
```

---

## 6. Verify Agents Are Connected

On the manager:
```bash
/var/ossec/bin/agent_control -l
```

Or in the Wazuh dashboard: **Agents → All agents** — all should show green/active.

---

## 7. Add Custom Decoder and Rules

See:
- [`configurations/local_decoder.xml`](../configurations/local_decoder.xml) — EdgeRouter decoder
- [`configurations/local_rules.xml`](../configurations/local_rules.xml) — custom detection rules

After adding/editing:
```bash
# Test config
/var/ossec/bin/wazuh-logtest

# Restart manager
systemctl restart wazuh-manager
```

---

## Troubleshooting

**Check manager logs:**
```bash
tail -f /var/ossec/logs/ossec.log
```

**Test a log line manually:**
```bash
/var/ossec/bin/wazuh-logtest
# Paste a sample log line, it will show which decoder and rule matched
```

**Agent not connecting:**
```bash
# On the agent
tail -f /var/ossec/logs/ossec.log

# Check manager is reachable
telnet 192.168.x.x 1514
```
