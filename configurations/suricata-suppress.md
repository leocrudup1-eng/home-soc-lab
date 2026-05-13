# Suricata Suppression Rules

Suppression rules reduce false positive noise while preserving detection capability. Each suppression below includes the rationale and what it preserves.

---

## Suppression File Location

`/etc/suricata/threshold.config`

---

## Current Suppression Rules

### SURICATA STREAM Errors
```
suppress gen_id 1, sig_id 2210051
suppress gen_id 1, sig_id 2210052
suppress gen_id 1, sig_id 2210053
```
**Reason:** SPAN/mirror port deployments by design see one-sided traffic, so TCP stream reassembly errors are expected and constant. These are structural noise from the passive capture method, not real threats.

**What's preserved:** Actual TCP-based attack alerts remain active.

---

### ET POLICY Tor Usage
```
suppress gen_id 1, sig_id 2522690, track by_src, ip 127.0.0.1
```
**Reason:** Tor was intentionally running on localhost during a lab session. Suppressed only for `127.0.0.1` — not globally — so unexpected Tor from other hosts still alerts.

**What's preserved:** Tor detection on all other hosts remains active.

---

### ET INFO Outbound DNS Query
```
suppress gen_id 1, sig_id 2027758
```
**Reason:** Pi-hole generates thousands of legitimate DNS queries per day. ET INFO DNS rules are visibility tools for enterprise environments, not threat indicators in a home lab context.

**What's preserved:** DNS-based exfiltration rules (different signatures) remain active.

---

### ET FILE_SHARING pythonhosted.org
```
suppress gen_id 1, sig_id 2034604
```
**Reason:** `suricata-update` uses Python to download rule packages from `files.pythonhosted.org` (Python Software Foundation CDN). This is the rule updater doing its job.

**What's preserved:** File sharing alerts on actual threat infrastructure remain active.

---

## Suppression Methodology

Before suppressing any rule:

1. **Identify the source** — which device triggered it and why
2. **Verify legitimacy** — confirm the traffic is expected and benign
3. **Suppress precisely** — use `track by_src` with specific IPs where possible, not global suppression
4. **Document** — record the rationale here so future sessions have context

> **Principle:** Suppress the noise, not the detection category. A suppressed rule should target one specific FP scenario, not disable an entire class of detection.
