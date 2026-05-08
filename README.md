# EDA Demo — Dynatrace + ServiceNow + Slack

Demo repository for **Event-Driven Ansible (EDA)** using the Dynatrace plugin. When Dynatrace detects a problem, EDA automatically creates a ServiceNow incident, attempts remediation, and posts real-time status updates to Slack — all without human intervention.

---

## Architecture

```
Dynatrace OneAgent
      │
      ▼  (problem detected)
Dynatrace Problems API
      │
      ▼  (polled by EDA every N seconds)
EDA Controller ──► Rulebook Activation
      │
      ├─ OPEN event ──► AAP Job Template ──► dt-alert-open.yml
      │                                          ├── Create ServiceNow Incident
      │                                          ├── Slack: 🚨 Alert (red)
      │                                          ├── Attempt auto-remediation
      │                                          ├── Update SNOW ticket
      │                                          ├── Slack: ✅ Resolved (green)
      │                                          └── Slack: 🆘 Escalate (red) if failed
      │
      └─ RESOLVED event ──► AAP Job Template ──► dt-alert-resolved.yml
                                                     ├── Look up open SNOW ticket
                                                     ├── Close SNOW ticket
                                                     └── Slack: 🟢 Auto-resolved
```

---

## Repository Structure

```
eda-demo/
├── rulebooks/
│   ├── dynatrace-snow-slack.yml   # Main rulebook: OPEN + RESOLVED rules (Slack integrated)
│   ├── dynatrace.yml              # Original rulebook: nginx process monitor only
│   └── dynatrace-debug.yml        # Debug rulebook: prints raw event payload
└── playbooks/
    ├── dt-alert-open.yml          # Full flow: SNOW + Slack + remediation
    ├── dt-alert-resolved.yml      # DT self-resolve: close SNOW + Slack notify
    ├── restart-nginx.yml          # Original: SNOW create/update/close + nginx restart
    └── simplenginx.yml            # Simple nginx restart only
```

---

## Prerequisites

### 1. Dynatrace Access Token

Generate a token in Dynatrace with the following scopes:

**API v1:**
- Access problem and event feed, metrics, and topology
- Read configuration / Write configuration

**API v2:**
- Read Problems / Write Problems
- Read Security Problems / Write Security Problems

### 2. Dynatrace OneAgent + Process Monitor

Install the OneAgent on your target RHEL host, then create a **Process Availability** monitoring rule:

1. Dynatrace → Settings → Process availability → Add monitoring rule
2. Name: `Nginx Host Monitor`
3. Detection rule → Property: `Executable`, Condition: `$contains(nginx)`

### 3. Decision Environment (EDA Controller)

Build a custom DE using `ansible-builder v3`:

```yaml
---
version: 3
images:
  base_image:
    name: registry.redhat.io/ansible-automation-platform-24/de-minimal-rhel8:latest
dependencies:
  galaxy:
    collections:
      - ansible.eda
      - dynatrace.event_driven_ansible
  system:
    - pkgconf-pkg-config [platform:rpm]
    - systemd-devel [platform:rpm]
    - gcc [platform:rpm]
    - python39-devel [platform:rpm]
options:
  package_manager_path: /usr/bin/microdnf
```

Push the DE to your Automation Hub or container registry, then import it into EDA Controller.

### 4. Execution Environment (AAP Controller)

Build or use an EE that includes the `servicenow.itsm` collection.

### 5. ServiceNow Custom Credential Type

In AAP Controller → Credential Types → Create:

**Input Configuration:**
```yaml
fields:
  - id: instance
    type: string
    label: Instance
  - id: username
    type: string
    label: Username
  - id: password
    type: string
    label: Password
    secret: true
required:
  - instance
  - username
  - password
```

**Injector Configuration:**
```yaml
env:
  SN_HOST: '{{ instance }}'
  SN_PASSWORD: '{{ password }}'
  SN_USERNAME: '{{ username }}'
```

### 6. Slack

Create a Slack app with a bot token (`xoxb-...`) scoped to `chat:write`. Note your target channel name.

---

## AAP Controller Setup

### Job Templates Required

| Template Name | Playbook | Credentials Needed |
|---|---|---|
| `EDA - DT Alert Open` | `playbooks/dt-alert-open.yml` | Machine, ServiceNow, Slack (extra vars) |
| `EDA - DT Alert Resolved` | `playbooks/dt-alert-resolved.yml` | ServiceNow, Slack (extra vars) |
| `Fix Nginx and update all` | `playbooks/restart-nginx.yml` | Machine, ServiceNow |

### Extra Variables (survey or job template vars)

```yaml
slack_token: xoxb-your-slack-bot-token
slack_channel: "#ansible-alerts"
dynatrace_host: https://xxxxx.live.dynatrace.com
```

---

## EDA Controller Setup

### Rulebook Activation Variables

Set the following as extra vars when creating the Rulebook Activation:

```yaml
dynatrace_host: https://xxxxx.live.dynatrace.com
dynatrace_token: YOUR_DYNATRACE_TOKEN
dynatrace_delay: 30
```

> **Note:** As of EDA Controller 2.x, rulebook variables cannot be vaulted. Store secrets in AAP Controller credentials where possible.

### Rulebook to Use

Select `rulebooks/dynatrace-snow-slack.yml` for the full Slack-integrated demo.  
Use `rulebooks/dynatrace-debug.yml` to inspect the raw event payload during setup.

---

## Running the Demo

Open four browser tabs:

| Tab | URL |
|---|---|
| 1 | ServiceNow — Incidents screen |
| 2 | Dynatrace — Problems screen |
| 3 | AAP Controller — Jobs screen |
| 4 | EDA Controller — Rule Audit screen |

Then kill nginx on your managed host:

```bash
sudo systemctl stop nginx
# or kill the process directly:
sudo kill -9 $(pgrep nginx)
```

Watch the chain fire automatically:
1. Dynatrace detects the problem (30–60 seconds)
2. EDA fires the rulebook rule
3. AAP job starts — ServiceNow ticket opens
4. Slack posts a red alert with the ticket number
5. Ansible attempts `nginx` restart
6. ServiceNow ticket closes, Slack posts green resolution

---

## Related Resources

- [EDA Rulebook Variables](https://ansible.readthedocs.io/projects/rulebook/en/stable/variables.html)
- [Dynatrace EDA Collection](https://galaxy.ansible.com/dynatrace/event_driven_ansible)
- [servicenow.itsm Collection](https://galaxy.ansible.com/servicenow/itsm)
- [Instruqt EDA Lab](https://play.instruqt.com/redhat/invite/g0wofhztypx3)
