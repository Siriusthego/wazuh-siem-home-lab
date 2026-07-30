# Wazuh Home Lab Troubleshooting – Windows Agent Disconnected

## Objective

Restore connectivity between a Windows 10 endpoint and a Wazuh Manager after the agent appeared as disconnected in the Wazuh environment.

---

## Initial Environment

### Wazuh Server (Ubuntu)

* Wazuh Manager
* Wazuh Indexer
* Wazuh Dashboard

### Endpoint

* Windows 10
* Wazuh Agent

---

## Problem Identification

After accessing the Ubuntu server, the agent status was checked using:

```bash
sudo /var/ossec/bin/agent_control -l
```

Output:

```text
ID: 000, Name: 220609030 (server), Active/Local
ID: 001, Name: Windows_Ev, Disconnected
```

The Windows endpoint existed in the Wazuh Manager inventory but was no longer connected.

---

## Investigation Process

### Step 1 – Verify Wazuh Services

To ensure the issue was not caused by a failed Wazuh component, service status was checked.

```bash
sudo systemctl status wazuh-manager
sudo systemctl status wazuh-dashboard
sudo systemctl status wazuh-indexer
```

Results:

* Wazuh Manager: Active
* Wazuh Dashboard: Active
* Wazuh Indexer: Active

Conclusion:

The Wazuh infrastructure was healthy and operational.

---

### Step 2 – Recover Dashboard Credentials

The dashboard password had been forgotten.

A search was performed for Wazuh password management utilities.

```bash
sudo find / -name "*password*" 2>/dev/null | grep wazuh
```

This revealed:

```text
/usr/share/wazuh-indexer/plugins/opensearch-security/tools/wazuh-passwords-tool.sh
```

The following command was executed:

```bash
sudo /usr/share/wazuh-indexer/plugins/opensearch-security/tools/wazuh-passwords-tool.sh -u admin
```

Result:

The existing dashboard administrator credentials were retrieved successfully, restoring dashboard access.

---

### Step 3 – Verify Endpoint Connectivity

On the Windows system:

```powershell
Get-Service WazuhSvc
```

Result:

```text
Status : Running
```

Conclusion:

The Wazuh Agent service was operational.

---

### Step 4 – Verify Network Reachability

The Ubuntu server IP address was identified:

```bash
hostname -I
```

Output:

```text
192.168.1.6
```

Connectivity from Windows to Ubuntu was tested:

```cmd
ping 192.168.1.6
```

Result:

Successful replies received.

Conclusion:

Network connectivity between endpoint and manager existed.

---

### Step 5 – Review Agent Configuration

The Wazuh Agent configuration file was examined:

```text
C:\Program Files (x86)\ossec-agent\ossec.conf
```

Relevant section:

```xml
<client>
  <server>
    <address>192.168.1.4</address>
    <port>1514</port>
    <protocol>tcp</protocol>
  </server>
</client>
```

Finding:

The agent was configured to communicate with:

```text
192.168.1.4
```

However, the current Wazuh Manager address was:

```text
192.168.1.6
```

Root Cause:

The Ubuntu server IP address had changed, but the Windows Agent configuration still pointed to the previous address.

---

## Remediation

The server address in ossec.conf was updated.

Before:

```xml
<address>192.168.1.4</address>
```

After:

```xml
<address>192.168.1.6</address>
```

The Wazuh Agent service was restarted:

```powershell
Restart-Service WazuhSvc
```

---

## Validation

Agent status was checked again from the Wazuh Manager.

```bash
sudo /var/ossec/bin/agent_control -l
```

Output:

```text
ID: 001, Name: Windows_Ev, Active
```

Result:

The Windows endpoint successfully reconnected to the Wazuh Manager.

---

## Lessons Learned

* Always verify core services before troubleshooting endpoint issues.
* Confirm network connectivity before investigating application-layer problems.
* Review endpoint configuration files when communication failures occur.
* DHCP or IP address changes can cause agents to lose connectivity.
* Wazuh password recovery utilities can be used to regain dashboard access without reinstalling the platform.

---

## Troubleshooting Methodology Used

1. Confirm infrastructure health.
2. Verify endpoint service status.
3. Test network connectivity.
4. Review configuration files.
5. Identify root cause.
6. Apply corrective action.
7. Validate successful recovery.

This approach follows a structured SOC troubleshooting workflow and helps quickly isolate connectivity issues within SIEM environments.

