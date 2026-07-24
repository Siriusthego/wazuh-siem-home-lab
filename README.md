# Wazuh SIEM Home Lab — Phase 1: Deployment, Endpoint Monitoring & Incident Detection

## Overview

This project documents the build-out of a self-hosted Wazuh SIEM environment, from initial deployment through endpoint agent integration, Sysmon-based telemetry collection, a live attack simulation, and vulnerability management. The focus of this phase was standing up a working detection pipeline end-to-end and documenting every failure encountered along the way — because in a real SOC, the troubleshooting is the job.

**Stack:** Ubuntu 22.04 (Wazuh Manager/Indexer/Dashboard), Windows 10 (monitored endpoint), Sysmon, PowerShell

> **Note:** This is Phase 1 of the lab. It covers SIEM deployment, agent onboarding, and a basic attack simulation. Phase 2 (in a separate repo/branch) adds a Kali Linux attack host, SSH brute-force and port-scan simulations, custom Wazuh/Sigma detection rules, and full MITRE ATT&CK technique mapping.

---

## 1. SIEM Deployment

**Goal:** Stand up a central log collection and alerting platform.

### Attempt 1 — Docker deployment
```bash
git clone -b v4.12.0 https://github.com/wazuh/wazuh-docker.git
cd wazuh-docker/single-node
docker compose up -d
```

This failed. The dashboard container entered a crash loop with:
```
PR_END_OF_FILE_ERROR
curl: SSL handshake failed
Connection refused (5601)
```

Digging into the container logs (`docker logs single-node_wazuh.dashboard_1`) revealed the actual cause:
```
EISDIR: illegal operation on a directory, read
root-ca.pem is a directory
```

The SSL certificate file had been mounted as a directory instead of a file, which broke the OpenSearch security plugin and put the indexer into a permanent restart loop. Manual cert cleanup attempts (`find . -name "*.pem" -type d -exec rm -rf {} \;`) hit permission errors due to the Docker volume mount, and a full stack reset (`docker compose down -v`, `docker system prune -af`, `docker volume prune -f`) still left the environment in a broken state. On top of that, a separate `docker-compose` vs `docker compose` CLI mismatch (`unknown shorthand flag: 'f'`) added friction to every recovery attempt.

**Decision:** After multiple failed recovery attempts, abandoned the Docker path in favor of the bare-metal installer.

### Attempt 2 — Bare-metal installer
```bash
curl -sO https://packages.wazuh.com/4.7/wazuh-install.sh
sudo bash wazuh-install.sh -a
```

This also failed initially:
```
ERROR: Cannot install dependency: systemd
```

Root cause analysis traced this back to a broken `apt` state, caused by a leftover ROS2 repository with conflicting GPG keys:
```
Conflicting values set for option Signed-By
```

Two different keyrings had been registered for the same ROS2 repo at some point, which corrupted `apt update` and blocked any dependency installation — including systemd, which the Wazuh installer needed.

**Fix:**
```bash
sudo rm -f /etc/apt/sources.list.d/ros2.list
sudo rm -f /etc/apt/sources.list.d/ros2.sources
sudo rm -f /usr/share/keyrings/ros-archive-keyring.gpg
sudo apt clean
sudo apt update
```

With `apt` repaired, the installer ran cleanly:
```bash
curl -sO https://packages.wazuh.com/4.10/wazuh-install.sh
sudo bash wazuh-install.sh -a
```

**Result:** Wazuh Manager, Indexer, and Dashboard up and running; dashboard login functional.

### Root cause summary
| Symptom | Actual cause |
|---|---|
| Dashboard crash loop / SSL handshake failure | `root-ca.pem` mounted as a directory, breaking the security plugin |
| `systemd` dependency install failure | Broken `apt` state |
| Broken `apt` state | Conflicting GPG keys from a leftover ROS2 repo |
| `401 Unauthorized` on dashboard login | Indexer security plugin never fully initialized due to the above |

**Takeaway:** The hardest part of running a SIEM isn't the SIEM itself — it's Linux dependency management and state consistency underneath it.

---

## 2. Endpoint Onboarding — Windows Agent

Connected a Windows 10 host as a monitored endpoint using the standard Wazuh agent installer via PowerShell (`Invoke-WebRequest`-based install command). Needed to run this from an elevated PowerShell session — running it from CMD or a non-admin shell silently failed. Once installed and started correctly, the agent showed as **Active** on the manager dashboard.

---

## 3. Deep Telemetry — Sysmon Integration

Default Windows Event Logs weren't detailed enough for meaningful investigation, so Sysmon was installed for process-level visibility:

```powershell
.\sysmon64.exe -i sysmonconfig-export.xml -accepteula
```

Initial run failed due to the config file not being found in the expected path and needing an elevated session; resolved by placing the XML alongside the executable and re-running as admin. Confirmed via **"Sysmon Started"** in the service output.

### Wiring Sysmon into Wazuh

Edited the agent's `ossec.conf` to forward Sysmon's operational log channel:

```xml
<localfile>
  <location>Microsoft-Windows-Sysmon/Operational</location>
  <log_format>eventchannel</log_format>
</localfile>
```

First attempt failed silently — the block had been placed outside the `</ossec_config>` closing tag, so it was never parsed. Moving it inside the tag fixed log ingestion immediately.

**Why `eventchannel` and not `syslog`:** Sysmon logs live in Windows's modern Event Log channel format. Using `syslog` as the format would have meant Wazuh couldn't parse Windows-native logs at all.

---

## 4. Attack Simulation & Detection

To validate the pipeline end-to-end, simulated unauthorized account creation on the monitored Windows host:

```powershell
net user Hacker_Kanka Sifre123! /add
```

**Result:** Wazuh fired a **Level 8 alert (Rule 60109)** for the account creation. A follow-up PowerShell action on the same host triggered a **Level 15 alert (Rule 92213)** — the highest Wazuh severity tier, indicating a rule set tuned to flag privilege-relevant PowerShell activity.

This confirmed the full pipeline — Sysmon → agent → manager → correlation rules → dashboard alert — was working correctly.

---

## 5. Vulnerability Management

Ran Wazuh's vulnerability detection module against the Windows endpoint and found 5 High and 6 Medium severity findings, largely tied to outdated Python and its dependencies. Remediated via:

```powershell
python.exe -m pip install --upgrade pip
pip install --upgrade urllib3 requests
```

Re-scanned and confirmed zero outstanding findings.

---

## Operational Reference

**Ubuntu (manager side):**
```bash
sudo systemctl status wazuh-manager
sudo systemctl start wazuh-manager wazuh-indexer wazuh-dashboard
sudo systemctl restart wazuh-manager
hostname -I
```

**Windows (agent side, run as Administrator):**
```powershell
Restart-Service Wazuh
Get-Service Sysmon64
Get-Service Wazuh
```
Key paths: config at `C:\Program Files (x86)\ossec-agent\ossec.conf`, agent logs at `C:\Program Files (x86)\ossec-agent\ossec.log`.

**If an agent shows "Disconnected":**
1. Confirm the manager's IP hasn't changed (`hostname -I`)
2. From the Windows host, `ping` the manager IP
3. Check the agent service status (`Get-Service Wazuh`)
4. Restart both sides if the above don't resolve it

---

## Skills Demonstrated

- SIEM deployment and Linux dependency/state troubleshooting (Docker and bare-metal)
- Endpoint agent deployment and Sysmon-based EDR-style telemetry
- Log pipeline configuration (`ossec.conf`, event channel routing)
- Alert triage and rule-severity interpretation
- Vulnerability detection and patch management

## What's next (Phase 2)

- Kali Linux attack host added to the lab
- SSH brute-force and port-scan simulations against the Windows/Linux endpoints
- Custom Wazuh rules and Sigma-format detection rules for the above
- Full MITRE ATT&CK technique mapping per simulated attack
