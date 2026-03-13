# Cybersecurity Home Lab — Atomic Red Team + Auditd + Wazuh SIEM

A complete adversary emulation and detection validation lab using Atomic Red Team, auditd, and Wazuh SIEM mapped to the MITRE ATT&CK framework.

---

## Lab Architecture

| VM | Role | OS |
|---|---|---|
| VM 1 | Wazuh Server (SIEM) | Ubuntu LTS |
| VM 2 | Atomic Red Team + Auditd (Agent) | Ubuntu LTS |

**Hypervisor:** VMware Workstation
**Network Mode:** NAT

---

## Prerequisites & Common Issues



## Part 1 — VM 1: Wazuh Server Setup

### Step 1 — Set a Static IP

```bash
sudo nano /etc/netplan/00-installer-config.yaml
```

```yaml
network:
  version: 2
  ethernets:
    ens33:
      dhcp4: no
      addresses: [192.168.x.10/24]      # replace x with your subnet
      gateway4: 192.168.x.1
      nameservers:
        addresses: [8.8.8.8]
```

```bash
sudo netplan apply
ip a    # verify static IP is assigned
```

### Step 2 — Download Wazuh Installer

```bash
curl -sO https://packages.wazuh.com/4.7/wazuh-install.sh
curl -sO https://packages.wazuh.com/4.7/config.yml
```

### Step 3 — Configure Wazuh

```bash
nano config.yml
```

Set all IPs to your Wazuh VM's static IP:

```yaml
nodes:
  indexer:
    - name: node-1
      ip: 192.168.x.10
  server:
    - name: wazuh-1
      ip: 192.168.x.10
  dashboard:
    - name: dashboard
      ip: 192.168.x.10
```

### Step 4 — Install Wazuh (All-in-One)

```bash
sudo bash wazuh-install.sh -a
```

> This takes 5–10 minutes. Save the **admin password** printed at the end.

### Step 5 — Enable Services on Boot

```bash
sudo systemctl enable wazuh-manager
sudo systemctl enable wazuh-indexer
sudo systemctl enable wazuh-dashboard
```

### Step 6 — Verify All Services Running

```bash
sudo systemctl status wazuh-manager
sudo systemctl status wazuh-indexer
sudo systemctl status wazuh-dashboard
```

### Step 7 — Access the Dashboard

Open a browser on your Windows host:

```
https://192.168.x.10
```

- **Username:** `admin`
- **Password:** printed at end of installation

> Browser will show a security warning (self-signed cert) — click **Advanced → Proceed**, this is normal.

---

## Part 2 — VM 2: Atomic Red Team + Auditd Setup

### Step 1 — Install Prerequisites

```bash
sudo apt update
sudo apt install -y curl git auditd
```

### Step 2 — Enable Auditd on Boot

```bash
sudo systemctl enable auditd
sudo systemctl start auditd
sudo systemctl status auditd
```

### Step 3 — Install PowerShell

```bash
curl https://packages.microsoft.com/keys/microsoft.asc | sudo apt-key add -
sudo curl -o /etc/apt/sources.list.d/microsoft.list https://packages.microsoft.com/config/ubuntu/22.04/prod.list
sudo apt update
sudo apt install -y powershell
```

### Step 4 — Install Atomic Red Team

Launch PowerShell:

```bash
pwsh
```

Inside PowerShell:

```powershell
IEX (IWR 'https://raw.githubusercontent.com/redcanaryco/invoke-atomicredteam/master/install-atomicredteam.ps1' -UseBasicParsing)

Install-AtomicRedTeam -getAtomics
```

### Step 5 — Auto-load Module on Every Session

```powershell
New-Item -Path $PROFILE -ItemType File -Force
Add-Content $PROFILE 'Import-Module "~/AtomicRedTeam/invoke-atomicredteam/Invoke-AtomicRedTeam.psd1"'
```

### Step 6 — Verify Atomic Red Team Works

```powershell
Import-Module "~/AtomicRedTeam/invoke-atomicredteam/Invoke-AtomicRedTeam.psd1"
Invoke-AtomicTest T1057 -ShowDetails
```

---

## Part 3 — Wazuh Agent Setup (VM 2)

### Step 1 — Install Wazuh Agent

```bash
curl -s https://packages.wazuh.com/key/GPG-KEY-WAZUH | sudo apt-key add -
echo "deb https://packages.wazuh.com/4.x/apt/ stable main" | sudo tee /etc/apt/sources.list.d/wazuh.list
sudo apt update
sudo apt install -y wazuh-agent
```

### Step 2 — Point Agent at Wazuh Server

```bash
sudo nano /var/ossec/etc/ossec.conf
```

Find and update:

```xml
<address>192.168.x.10</address>
```

### Step 3 — Forward Auditd Logs to Wazuh

In the same `ossec.conf`, add inside `<ossec_config>`:

```xml
<localfile>
  <log_format>audit</log_format>
  <location>/var/log/audit/audit.log</location>
</localfile>
```

### Step 4 — Enable and Start Agent

```bash
sudo systemctl enable wazuh-agent
sudo systemctl start wazuh-agent
```

---

## Part 4 — Register Agent on Wazuh Server

On **VM 1 (Wazuh Server)**:

```bash
sudo /var/ossec/bin/manage_agents
```

- Press `A` → Add agent
- Enter a name (e.g. `atomic-vm`)
- Enter the agent VM's IP
- Note the **key generated**

On **VM 2 (Agent VM)**:

```bash
sudo /var/ossec/bin/manage_agents
```

- Press `I` → Import key
- Paste the key from the server

Restart agent:

```bash
sudo systemctl restart wazuh-agent
```

---

## Part 5 — Running Atomic Tests

### Always preview a test before running

```powershell
Invoke-AtomicTest T1082 -ShowDetails
```

### Always clean up after

```powershell
Invoke-AtomicTest T1082 -Cleanup
```

### Recommended Test Sequence

#### Discovery (Safe — Start Here)

```powershell
Invoke-AtomicTest T1057   # Process Discovery
Invoke-AtomicTest T1082   # System Information Discovery
Invoke-AtomicTest T1083   # File & Directory Discovery
Invoke-AtomicTest T1033   # System Owner/User Discovery
Invoke-AtomicTest T1049   # System Network Connections
```

#### Defense Evasion

```powershell
Invoke-AtomicTest T1070.003  # Clear bash history
Invoke-AtomicTest T1070.002  # Clear syslog
Invoke-AtomicTest T1222.002  # chmod to hide files
```

#### Persistence

```powershell
Invoke-AtomicTest T1053.003  # Cron job persistence
Invoke-AtomicTest T1098       # Account manipulation
Invoke-AtomicTest T1136.001  # Create local user account
```

#### Credential Access

```powershell
Invoke-AtomicTest T1110.001  # Brute force SSH
Invoke-AtomicTest T1003.008  # /etc/passwd & /etc/shadow dump
```

#### Execution

```powershell
Invoke-AtomicTest T1059.004  # Unix shell execution
Invoke-AtomicTest T1059.006  # Python script execution
```

#### Exfiltration

```powershell
Invoke-AtomicTest T1048   # Exfiltration over alternative protocol
Invoke-AtomicTest T1041   # Exfiltration over C2 channel
```

---

## Part 6 — Viewing Logs

### Raw Auditd Logs

```bash
# Live log stream
sudo tail -f /var/log/audit/audit.log

# Recent events
sudo ausearch -ts recent

# Readable summary
sudo aureport -i

# Filter by process
sudo ausearch -c pwsh

# Filter by time window
sudo ausearch -ts -10min

# Syscall summary
sudo aureport --syscall -i
```

### Wazuh Dashboard

Open in browser on Windows host:

```
https://192.168.x.10
```

Key sections:
- **Security Events** → all alerts from agent
- **MITRE ATT&CK** → techniques heatmap
- **Integrity Monitoring** → file changes
- **Vulnerability Detection** → CVEs on agent VM

---

## Recommended Workflow Per Test

```bash
# 1. Note the time
date

# 2. Run the atomic (inside pwsh)
Invoke-AtomicTest T1082

# 3. Check auditd logs
sudo ausearch -ts recent

# 4. Verify alert appeared in Wazuh dashboard
# Security Events → filter by recent time

# 5. Clean up
Invoke-AtomicTest T1082 -Cleanup
```

---

## Quick Reference — Service Status Commands

### Wazuh Server VM

```bash
sudo systemctl status wazuh-manager
sudo systemctl status wazuh-indexer
sudo systemctl status wazuh-dashboard
```

### Atomic Red Team VM

```bash
sudo systemctl status wazuh-agent
sudo systemctl status auditd
```

---

## ATT&CK Coverage Map

| Tactic | Technique | Test ID |
|---|---|---|
| Discovery | Process Discovery | T1057 |
| Discovery | System Info Discovery | T1082 |
| Discovery | File & Directory Discovery | T1083 |
| Discovery | System Owner/User Discovery | T1033 |
| Discovery | Network Connections | T1049 |
| Defense Evasion | Clear Bash History | T1070.003 |
| Defense Evasion | Clear Syslog | T1070.002 |
| Defense Evasion | File Permissions Modification | T1222.002 |
| Persistence | Cron Job | T1053.003 |
| Persistence | Account Manipulation | T1098 |
| Persistence | Create Local Account | T1136.001 |
| Credential Access | Brute Force SSH | T1110.001 |
| Credential Access | OS Credential Dumping | T1003.008 |
| Execution | Unix Shell | T1059.004 |
| Execution | Python Execution | T1059.006 |
| Exfiltration | Alternative Protocol | T1048 |
| Exfiltration | Over C2 Channel | T1041 |

---

## Images 

<img width="1481" height="985" alt="Screenshot 2026-03-13 194345" src="https://github.com/user-attachments/assets/215207f8-23c6-498e-91b0-8e16fd310b93" />

---
<img width="1660" height="970" alt="Screenshot 2026-03-13 194406" src="https://github.com/user-attachments/assets/d83ff6ef-3e3e-42ff-aa63-9d371045e213" />

---
<img width="1855" height="859" alt="Screenshot 2026-03-13 194438" src="https://github.com/user-attachments/assets/fb5f34c5-1bb8-4c8a-9fa2-6390f355ca77" />

---
<img width="1855" height="880" alt="Screenshot 2026-03-13 194458" src="https://github.com/user-attachments/assets/6ae50be8-5e6b-478b-aaec-9d7c268a1013" />

---
<img width="1850" height="858" alt="Screenshot 2026-03-13 194517" src="https://github.com/user-attachments/assets/08000038-81c1-43d9-89f3-27511fef401a" />

---
<img width="1857" height="654" alt="Screenshot 2026-03-13 194613" src="https://github.com/user-attachments/assets/d8db4939-50fc-4187-a1f4-124d846ac6af" />








