# Lessons Learned

Running notes from building and operating this lab. Updated as new things break, surprise, or teach something useful.

---

## Architecture & Setup

### SPAN port is passive — that's a feature, not a bug
Putting Suricata on a mirror port means it sees all traffic without being inline. If Suricata crashes or the laptop reboots, the network keeps running. Always choose passive monitoring over inline for a home lab IDS.

### Custom decoders require exact regex — test before restarting
A bad Wazuh decoder will crash `wazuh-analysisd` and take the manager down. Always validate with:
```bash
/var/ossec/bin/wazuh-logtest
```
before `systemctl restart wazuh-manager`. Learn this the hard way once, never again.

### One dedicated device per critical function
Pi-hole runs only Pi-hole. DNS filtering should survive everything else going down. Mixing roles on one machine means one reboot takes down multiple functions.

---

## Alert Triage

### The first question is always: what generated this?
Before classifying any alert, identify the source device and what it was doing at the time. The same alert signature means completely different things depending on context.

### Suppress precisely, never broadly
When suppressing a false positive, use `track by_src` with a specific IP where possible. Suppressing globally can blind you to the same attack coming from an unexpected source.

Example: Tor was suppressed only for `127.0.0.1`. Tor from any other device still alerts.

### ET INFO rules are visibility tools, not threat detectors
`ET INFO` category rules fire on normal, legitimate activity. In enterprise environments they provide visibility into policy violations. In a home lab they generate noise. Understand what category a rule belongs to before deciding on suppression.

### Correlation is where triage gets interesting
A single alert is a data point. Two alerts from the same source IP 68 seconds apart form a pattern. Always check what else happened around the same time from the same source.

### SPAN port stream errors are structural — suppress them early
Mirror port deployments see one-sided traffic by definition. TCP stream reassembly errors will flood `fast.log` constantly. Suppress Suricata STREAM SID groups early and move on to alerts that matter.

### Document everything — even false positives
False positive documentation is valuable. It tells the next person (or future you) why a known-good alert was cleared, and prevents re-investigating the same thing repeatedly.

---

## Windows Monitoring

### Sysmon + Wazuh SCA creates its own noise
Wazuh's SCA module runs as SYSTEM on Windows boot and uses `secedit` to check password policy settings. Sysmon logs this as privileged PowerShell execution. This looks suspicious until you know what it is.

Rule of thumb: `NT AUTHORITY\SYSTEM` + `powershell secedit /export /cfg` on boot = Wazuh SCA. Document it and move on.

### Wazuh agent + Sysmon is the right combination for Windows
Wazuh agent alone on Windows gives you Event Log. Sysmon turns that into SOC-grade telemetry: full process command lines, network connections with process context, registry changes. Always deploy both.

---

## Operational

### Check `dmesg` first for hardware issues
When a drive starts throwing I/O errors, `dmesg` shows the kernel-level story before any application layer error makes sense of it. `dmesg | tail -30` is often the fastest path to understanding hardware failures.

### Load cycle count matters more than reallocated sectors on old drives
A drive with zero reallocated sectors but 470,000+ load cycles (rated for ~300,000) has a shorter remaining life than SMART's health assessment suggests. High LCC on USB drives often comes from aggressive power management — the drive parks its head excessively over USB.

### Backups to a single destination aren't backups
A backup that fails 5 times in a row to a single aging USB drive is a warning, not bad luck. Redundancy is the point.
