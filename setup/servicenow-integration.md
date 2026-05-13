# Wazuh + ServiceNow Integration

## Overview

Wazuh is configured to automatically create incident tickets in ServiceNow for any alert at rule level 12 or above. This bridges the gap between security detection and incident management — high-priority alerts flow directly into a structured ticketing workflow without manual intervention.

**Reference:** [Integrating ServiceNow with Wazuh](https://wazuh.com/blog/integrating-servicenow-with-wazuh/) (Wazuh Blog)

---

## Status

| Component | Status |
|---|---|
| ServiceNow developer instance | ✅ Created |
| REST API verified | ✅ Working |
| `.env` credentials configured | ✅ Done |
| Integration script deployed | ✅ Done |
| Level 12+ alerts → ServiceNow tickets | ✅ Working |
| ServiceNow email notifications on new ticket | ⚠️ In progress |

---

## Infrastructure

- **Wazuh Manager** — Ubuntu Server 24.04 laptop
- **ServiceNow** — Free personal developer instance (ServiceNow Developer Program)

> ServiceNow does not offer a completely free public API. A free personal developer instance is available at https://developer.servicenow.com — full-featured for testing and home lab use.

---

## 1. ServiceNow Developer Instance Setup

### Create account and request instance

1. Go to https://developer.servicenow.com
2. Sign up and verify email
3. Log in → click **Request Instance** (top-right)
4. Choose instance type and click **Request**
5. Your instance URL will be: `https://devXXXXX.service-now.com`
6. Click **Manage instance password** to retrieve credentials
7. Change user role to **Admin**

### Verify REST API access

From the instance dashboard:
1. Click **All** → search **REST API Explorer**
2. Set: Namespace: `now`, API Name: `Table API`, Version: `latest`
3. Click **Create a record (POST)**
4. Set `tableName` to `incident`
5. In the **RAW** request body, paste:

```json
{
  "short_description": "Wazuh Alert - Test",
  "urgency": "2",
  "impact": "2"
}
```

6. Click **Send** — a `200` response with an `INC` number confirms API access is working

---

## 2. Wazuh Server Configuration

### Create the `.env` credentials file

```bash
sudo nano /var/ossec/integrations/.env
```

Add:
```
SERVICENOW_USER=<your_username>
SERVICENOW_PASS=<your_password>
SERVICENOW_INSTANCE=<your_instance>.service-now.com
```

Set permissions:
```bash
sudo chmod 750 /var/ossec/integrations/.env
sudo chown root:wazuh /var/ossec/integrations/.env
```

> **Security note:** Credentials are stored in a `.env` file rather than hardcoded in the script. The file is owned by root:wazuh with restricted permissions. Never commit the `.env` file to version control.

---

### Install Python dependencies

```bash
sudo /var/ossec/framework/python/bin/pip3 install requests python-dotenv
```

---

### Deploy the integration script

Create `/var/ossec/integrations/custom-servicenow`:

```python
#!/var/ossec/framework/python/bin/python3.10
# Wazuh -> ServiceNow integration script

import json
import sys
import time
import os

try:
    import requests
    from requests.auth import HTTPBasicAuth
    from dotenv import load_dotenv
except Exception as e:
    print("Required modules not found. Install with: pip install requests python-dotenv")
    sys.exit(1)

# Load .env from same directory as script
script_dir = os.path.dirname(os.path.realpath(__file__))
dotenv_path = os.path.join(script_dir, '.env')
load_dotenv(dotenv_path)

SN_INSTANCE = os.getenv('SERVICENOW_INSTANCE')
SN_USER = os.getenv('SERVICENOW_USER')
SN_PASS = os.getenv('SERVICENOW_PASS')
SN_TABLE_URL = f"https://{SN_INSTANCE}/api/now/table/incident"

debug_enabled = True
pwd = os.path.dirname(os.path.dirname(os.path.realpath(__file__)))
now = time.strftime("%a %b %d %H:%M:%S %Z %Y")
log_file = '{0}/logs/integrations.log'.format(pwd)

def main(args):
    debug("# Starting")
    if not SN_INSTANCE or not SN_USER or not SN_PASS:
        debug("Missing ServiceNow configuration in environment variables.")
        sys.exit(1)

    alert_file_location = args[1]
    with open(alert_file_location, 'rb') as alert_file:
        last_line = alert_file.read().decode('utf-8').splitlines()[-1]
        if last_line.split():
            json_alert = json.loads(last_line)

    msg = generate_payload(json_alert)
    send_msg(msg)

def debug(msg):
    if debug_enabled:
        msg = "{0}: {1}\n".format(now, msg)
        print(msg)
        with open(log_file, "a") as f:
            f.write(msg)

def generate_payload(alert):
    title = alert['rule'].get('description', "No Description")
    alert_level = str(alert['rule'].get('level', "N/A"))
    agentname = alert['agent'].get('name', "No Name")
    agentip = alert['agent'].get('ip', "No IP")
    location = alert.get('location', "No Location")
    timestamp = alert.get('timestamp', "No Timestamp")
    full_log = alert.get('full_log', "No Log")

    return {
        "short_description": f"Wazuh Alert: {title}",
        "description": f"Timestamp: {timestamp}\nAgent: {agentname} ({agentip})\nLevel: {alert_level}\nLocation: {location}\nLog: {full_log}",
        "urgency": "1",
        "impact": "1",
        "category": "Security"
    }

def send_msg(payload):
    try:
        response = requests.post(
            SN_TABLE_URL,
            auth=HTTPBasicAuth(SN_USER, SN_PASS),
            headers={"Content-Type": "application/json"},
            json=payload
        )
        debug(f"ServiceNow response: {response.status_code} {response.text}")
    except Exception as e:
        debug(f"Error sending to ServiceNow: {e}")

if __name__ == "__main__":
    try:
        if len(sys.argv) >= 2:
            msg = '{0} {1}'.format(now, sys.argv[1])
        else:
            debug("# Exiting: Bad arguments.")
            sys.exit(1)
        with open(log_file, 'a') as f:
            f.write(msg + '\n')
        main(sys.argv)
    except Exception as e:
        debug(str(e))
        raise
```

Set permissions:
```bash
sudo chmod 750 /var/ossec/integrations/custom-servicenow
sudo chown root:wazuh /var/ossec/integrations/custom-servicenow
```

---

### Configure Wazuh integrator module

Add to `/var/ossec/etc/ossec.conf` inside `<ossec_config>`:

```xml
<!-- ServiceNow Integration -->
<integration>
  <name>custom-servicenow</name>
  <hook_url>https://<SN_INSTANCE>.service-now.com/api/now/table/incident</hook_url>
  <level>12</level>
  <alert_format>json</alert_format>
</integration>
```

Replace `<SN_INSTANCE>` with your actual instance name (e.g. `dev12345`).

Restart Wazuh manager:
```bash
sudo systemctl restart wazuh-manager
```

---

## 3. Verify the Integration

Check the integration log on the Wazuh manager:
```bash
tail -f /var/ossec/logs/integrations.log
```

Trigger a level 12 alert (e.g. failed Windows login attempts) and confirm:
1. Alert appears in Wazuh dashboard under **Threat Intelligence → Threat Hunting**
2. Incident ticket created in ServiceNow under **Service Desk → Incidents**

---

## 4. Pending — Email Notifications from ServiceNow

ServiceNow email notifications on new incident creation are not yet configured. The goal is for ServiceNow to send an email when a Wazuh alert creates a new ticket.

**To configure (in progress):**
In the ServiceNow instance:
1. Go to **All → Notifications**
2. Create a new notification triggered on **Incident → Inserted**
3. Set conditions (e.g. Category = Security, State = New)
4. Configure email recipients and template

---

## Architecture Impact

```
Wazuh Manager
    │
    ├── Rule level ≥ 12 alert fires
    │
    └── Integrator module
            │
            └── custom-servicenow script
                    │
                    ├── Loads credentials from .env
                    └── POST → ServiceNow REST API
                                    │
                                    └── Incident ticket created
                                            │
                                            └── (pending) Email notification sent
```

---

## Security Notes

- Basic Auth (username/password) is used for this developer instance
- For production: OAuth token is recommended
- `.env` file is permission-restricted (`750`, `root:wazuh` ownership)
- `.env` must be added to `.gitignore` — never commit credentials
