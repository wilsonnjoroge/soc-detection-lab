# SOC Detection Lab — Setup Guide

**Document Type:** Technical Setup Reference
**Companion To:** README.md, siem_lab_runbook.md

> Step-by-step installation and configuration guide for all lab components. Use placeholders where shown — replace with your actual values from your lab environment.

---

## Table of Contents

1. [Environment Preparation](#1-environment-preparation)
2. [Wazuh All-in-One Deployment](#2-wazuh-all-in-one-deployment)
3. [Wazuh Agent — Linux](#3-wazuh-agent--linux)
4. [Wazuh Agent — Windows](#4-wazuh-agent--windows)
5. [Suricata Network IDS](#5-suricata-network-ids)
6. [Suricata to Wazuh Integration](#6-suricata-to-wazuh-integration)
7. [Log Collection Configuration](#7-log-collection-configuration)
8. [Custom Detection Rules](#8-custom-detection-rules)
9. [Detection Improvements](#9-detection-improvements)
10. [Troubleshooting](#10-troubleshooting)

---

## Placeholder Reference

| Placeholder | Description |
|---|---|
| `<WAZUH_IP>` | Wazuh VM IP |
| `<KALI_IP>` | Kali Linux attacker IP |
| `<METASPLOITABLE3_UBUNTU_IP>` | Metasploitable3 Ubuntu target IP |
| `<WINDOWS_11_IP>` | Windows 11 target IP |
| `<WINSERVER_2008_IP>` | Windows Server 2008 target IP |
| `<HOST_IP>` | Host machine IP |

---

## 1. Environment Preparation

### Network

All VMs connect to a VMware Host-Only network (`192.168.172.0/24`). VMs requiring internet access for package downloads use an additional NAT adapter on a per-VM basis.

Each VM needs:
- Adapter 1: VMware Host-Only (lab traffic)
- Adapter 2: NAT (internet — for package downloads only, where needed)

### Verify Connectivity

```bash
# From Wazuh VM — confirm all targets are reachable
ping -c 3 <KALI_IP>
ping -c 3 <METASPLOITABLE3_UBUNTU_IP>
ping -c 3 <WINDOWS_11_IP>

# Confirm Wazuh ports are listening
sudo ss -tlnp | grep -E '1514|1515|9200|443'
```

---

## 2. Wazuh All-in-One Deployment

Deploys Wazuh Manager, Indexer, Dashboard, and Filebeat on the Wazuh VM in a single automated process.

### Step 1 — Prepare the VM

```bash
sudo apt update && sudo apt upgrade -y
sudo hostnamectl set-hostname wazuh-siem
```

### Step 2 — Download Installation Assets

```bash
mkdir -p ~/Desktop/wazuh && cd ~/Desktop/wazuh
curl -sO https://packages.wazuh.com/4.x/wazuh-install.sh
curl -sO https://packages.wazuh.com/4.x/config.yml
```

### Step 3 — Configure Node IPs

Edit `config.yml` and set all node IPs to `<WAZUH_IP>`:

```yaml
nodes:
  indexer:
    - name: node-1
      ip: "<WAZUH_IP>"
  server:
    - name: wazuh-1
      ip: "<WAZUH_IP>"
  dashboard:
    - name: dashboard
      ip: "<WAZUH_IP>"
```

### Step 4 — Run Installer

```bash
sudo bash wazuh-install.sh -a
```

The installer will complete the following stages:
1. Install dependencies and add Wazuh repository
2. Generate all TLS certificates
3. Install and start Wazuh Indexer — initialise cluster security
4. Install and start Wazuh Manager + Filebeat
5. Install and start Wazuh Dashboard

> Save the admin credentials printed at the end of installation.

### Step 5 — Verify Services

```bash
sudo systemctl status wazuh-manager wazuh-indexer wazuh-dashboard filebeat
```

All four must show `active (running)`.

### Step 6 — Access Dashboard

Navigate to `https://<WAZUH_IP>` in a browser.
- Username: `admin`
- Password: from installer output

Confirm all health checks pass on the dashboard home screen.

---

## 3. Wazuh Agent — Linux

Applies to: Kali Linux, Metasploitable3 Ubuntu, Host Ubuntu.

### Step 1 — Download and Install

```bash
wget https://packages.wazuh.com/4.x/apt/pool/main/w/wazuh-agent/wazuh-agent_4.14.5-1_amd64.deb

sudo WAZUH_MANAGER='<WAZUH_IP>' \
     WAZUH_AGENT_GROUP='linux-agents' \
     WAZUH_AGENT_NAME='<agent-name>' \
     dpkg -i ./wazuh-agent_4.14.5-1_amd64.deb
```

### Step 2 — Start Agent

```bash
sudo systemctl daemon-reload
sudo systemctl enable wazuh-agent
sudo systemctl start wazuh-agent
```

### Step 3 — Verify Connection

```bash
sudo tail -f /var/ossec/logs/ossec.log
# Look for: INFO: (4102): Connected to the server ([<WAZUH_IP>]:1514/tcp)
```

### Step 4 — Confirm on Manager

```bash
sudo /var/ossec/bin/agent_control -la
```

### Troubleshooting — Wrong Manager IP

If the agent was installed with an incorrect manager IP:

```bash
sudo nano /var/ossec/etc/ossec.conf
# Find <server> block and set:
# <address><WAZUH_IP></address>

sudo systemctl restart wazuh-agent
sudo grep "Connected\|ERROR" /var/ossec/logs/ossec.log | tail -10
```

---

## 4. Wazuh Agent — Windows

Applies to: Windows 11, Windows Server 2008.

### Option A — PowerShell

```powershell
# Run as Administrator
Invoke-WebRequest -Uri "https://packages.wazuh.com/4.x/windows/wazuh-agent-4.14.5-1.msi" `
  -OutFile "$env:TEMP\wazuh-agent.msi"

msiexec.exe /i "$env:TEMP\wazuh-agent.msi" /q `
  WAZUH_MANAGER="<WAZUH_IP>" `
  WAZUH_AGENT_NAME="<agent-name>" `
  WAZUH_AGENT_GROUP="windows-agents"

NET START WazuhSvc
```

### Option B — GUI Installer

1. Download MSI from `https://packages.wazuh.com/4.x/windows/`
2. Run installer
3. Set Manager IP: `<WAZUH_IP>`
4. Set Agent Name: `<agent-name>`
5. Complete installation and start service

### Verify Connection

```powershell
# Check service status
Get-Service WazuhSvc

# Check agent log
Get-Content "C:\Program Files (x86)\ossec-agent\ossec.log" -Tail 20
```

---

## 5. Suricata Network IDS

Installed on the Wazuh VM to monitor lab network traffic.

### Step 1 — Install

```bash
sudo add-apt-repository ppa:oisf/suricata-stable -y
sudo apt update
sudo apt install suricata -y
suricata --build-info | grep Version
```

### Step 2 — Update Rules

```bash
sudo suricata-update
```

### Step 3 — Identify Monitoring Interface

```bash
ip a
# Find the interface carrying <WAZUH_IP> — e.g. ens33, ens37, eth0
```

### Step 4 — Configure suricata.yaml

```bash
sudo nano /etc/suricata/suricata.yaml
```

Key settings:

```yaml
vars:
  address-groups:
    HOME_NET: "[192.168.172.0/24]"
    EXTERNAL_NET: "!$HOME_NET"

af-packet:
  - interface: <your-interface>
    cluster-id: 99
    cluster-type: cluster_flow
    defrag: yes

outputs:
  - eve-log:
      enabled: yes
      filetype: regular
      filename: /var/log/suricata/eve.json
      types:
        - alert:
            payload: yes
            packet: yes
            metadata: yes
        - dns
        - http:
            extended: yes
        - ssh
        - flow
```

### Step 5 — Start Suricata

```bash
sudo systemctl enable suricata
sudo systemctl start suricata
sudo systemctl status suricata
```

### Step 6 — Verify Traffic Capture

```bash
sudo tail -f /var/log/suricata/eve.json
```

Generate test traffic from another VM — events should appear within seconds.

---

## 6. Suricata to Wazuh Integration

### Step 1 — Edit ossec.conf

```bash
sudo nano /var/ossec/etc/ossec.conf
```

Add inside `<ossec_config>`:

```xml
<localfile>
  <log_format>json</log_format>
  <location>/var/log/suricata/eve.json</location>
</localfile>
```

### Step 2 — Set File Permissions

```bash
sudo chmod 644 /var/log/suricata/eve.json
sudo chown root:wazuh /var/log/suricata/eve.json
```

### Step 3 — Restart Manager

```bash
sudo systemctl restart wazuh-manager
```

### Step 4 — Verify

```bash
sudo grep -i suricata /var/ossec/logs/ossec.log
```

Then in Wazuh Dashboard: Security Events → filter by `suricata`.

---

## 7. Log Collection Configuration

Log collection in this lab is configured centrally using **Wazuh agent groups**. Rather than editing `ossec.conf` on each individual agent, configuration is pushed from the manager to all agents in the group automatically.

This is the recommended approach — changes are made once on the manager and applied to all group members immediately.

### How Agent Groups Work

```
Manager pushes config ──► agents in group pull it on next check-in
No manual edits needed on individual agent machines
```

Group configs live at:
```bash
/var/ossec/etc/shared/<group-name>/agent.conf
```

### Groups Used in This Lab

| Group | Members | Config Covers |
|---|---|---|
| `linux-agents` | kali-agent, metasploitable3-ubuntu-agent, host-ubuntu-agent | FIM, auth.log, syslog, apache logs |
| `windows-agents` | windows-11-agent | FIM, Windows Event Logs, Sysmon |

---

### Linux Agents Group Config

File: `/var/ossec/etc/shared/linux-agents/agent.conf`

```xml
<agent_config>
  <!-- ============================================================ -->
  <!-- FILE INTEGRITY MONITORING                                     -->
  <!-- ============================================================ -->
  <syscheck>
    <disabled>no</disabled>
    <frequency>300</frequency>
    <scan_on_start>yes</scan_on_start>
    <!-- Real-time monitoring: critical config directory -->
    <directories check_all="yes" realtime="yes">/etc</directories>
    <!-- Periodic monitoring: system binaries -->
    <directories check_all="yes">/usr/bin,/usr/sbin,/bin,/sbin</directories>
    <!-- Ignore noisy files that change frequently -->
    <ignore>/etc/mtab</ignore>
    <ignore>/etc/hosts.deny</ignore>
    <ignore>/etc/resolv.conf</ignore>
  </syscheck>

  <!-- ============================================================ -->
  <!-- LOG COLLECTION                                               -->
  <!-- ============================================================ -->
  <localfile>
    <log_format>syslog</log_format>
    <location>/var/log/auth.log</location>
  </localfile>

  <localfile>
    <log_format>syslog</log_format>
    <location>/var/log/syslog</location>
  </localfile>

  <localfile>
    <log_format>apache</log_format>
    <location>/var/log/apache2/access.log</location>
  </localfile>

  <localfile>
    <log_format>apache</log_format>
    <location>/var/log/apache2/error.log</location>
  </localfile>
</agent_config>
```

> Note: Apache log paths only apply to agents that have Apache installed (Metasploitable3 Ubuntu). Wazuh will silently skip log paths that do not exist on an agent.

---

### Windows Agents Group Config

File: `/var/ossec/etc/shared/windows-agents/agent.conf`

```xml
<agent_config>
  <!-- ============================================================ -->
  <!-- FILE INTEGRITY MONITORING                                     -->
  <!-- ============================================================ -->
  <syscheck>
    <disabled>no</disabled>
    <frequency>300</frequency>
    <scan_on_start>yes</scan_on_start>
    <!-- Monitor common persistence and execution paths -->
    <directories check_all="yes">%WINDIR%\System32</directories>
    <directories check_all="yes">%PROGRAMFILES%</directories>
    <directories check_all="yes" realtime="yes">%USERPROFILE%\Desktop</directories>
    <!-- Registry monitoring: common persistence locations -->
    <windows_registry>HKEY_LOCAL_MACHINE\Software\Microsoft\Windows\CurrentVersion\Run</windows_registry>
    <windows_registry>HKEY_LOCAL_MACHINE\Software\Microsoft\Windows\CurrentVersion\RunOnce</windows_registry>
    <windows_registry>HKEY_LOCAL_MACHINE\System\CurrentControlSet\Services</windows_registry>
  </syscheck>

  <!-- ============================================================ -->
  <!-- LOG COLLECTION — Windows Event Log                           -->
  <!-- ============================================================ -->
  <localfile>
    <log_format>eventchannel</log_format>
    <location>Security</location>
  </localfile>

  <localfile>
    <log_format>eventchannel</log_format>
    <location>System</location>
  </localfile>

  <localfile>
    <log_format>eventchannel</log_format>
    <location>Microsoft-Windows-Sysmon/Operational</location>
  </localfile>
</agent_config>
```

> Note: Sysmon must be installed separately on Windows agents for the Sysmon/Operational channel to be available.

---

### Applying Group Config Changes

After editing any `agent.conf`:

```bash
# Verify config is valid
sudo /var/ossec/bin/verify-agent-conf

# Restart manager to push changes
sudo systemctl restart wazuh-manager

# Confirm agents received the update
sudo /var/ossec/bin/agent_control -la
```

Agents pull the updated config on next check-in (within 60 seconds by default).

---

### FTP Logs (Metasploitable3 Ubuntu Only)

FTP logs are not included in the shared linux-agents group config because not all Linux agents run vsftpd. Add this directly to Metasploitable3's individual agent config if needed:

```bash
sudo nano /var/ossec/etc/ossec.conf   # on metasploitable3-ubuntu-agent
```

```xml
<localfile>
  <log_format>syslog</log_format>
  <location>/var/log/vsftpd.log</location>
</localfile>
```

Restart agent after:
```bash
sudo systemctl restart wazuh-agent
```

---

## 8. Custom Detection Rules

Custom rules are placed at `/var/ossec/etc/rules/local_rules.xml` on the Wazuh manager.

### Detect Nmap User-Agent in HTTP

```xml
<group name="web,nmap,">
  <rule id="100001" level="10">
    <if_group>web|accesslog</if_group>
    <match>Nmap Scripting Engine</match>
    <description>Nmap HTTP scan detected in web server logs</description>
    <mitre>
      <id>T1046</id>
    </mitre>
  </rule>
</group>
```

### SSH Brute Force — Custom Threshold

```xml
<group name="syslog,ssh,">
  <rule id="100002" level="12" frequency="6" timeframe="60">
    <if_matched_sid>5716</if_matched_sid>
    <description>SSH brute force: 6 or more failed attempts in 60 seconds</description>
    <mitre>
      <id>T1110</id>
    </mitre>
  </rule>
</group>
```

### Command Injection Pattern

```xml
<group name="web,injection,">
  <rule id="100003" level="14">
    <if_group>web|accesslog</if_group>
    <match>cat /etc|whoami|/bin/bash|/etc/passwd</match>
    <description>Command injection attempt detected in web logs</description>
    <mitre>
      <id>T1059</id>
    </mitre>
  </rule>
</group>
```

### Reload Rules

```bash
sudo /var/ossec/bin/wazuh-control restart
```

### Custom Suricata Rules

Add to `/etc/suricata/rules/local.rules`:

```bash
# Detect Hydra SSH brute force
alert tcp any any -> $HOME_NET 22 (msg:"HYDRA SSH Brute Force Detected"; \
  flow:to_server; threshold:type threshold, track by_src, count 5, seconds 60; \
  sid:9000001; rev:1;)

# Detect SQLMap scanner
alert http any any -> $HOME_NET any (msg:"SQLMap Scanner Detected"; \
  http.user_agent; content:"sqlmap"; nocase; \
  sid:9000002; rev:1;)

# Detect DVWA SQL injection
alert http any any -> $HOME_NET any (msg:"DVWA SQLi Attempt"; \
  http.uri; content:"SELECT"; nocase; \
  sid:9000003; rev:1;)
```

After adding rules:

```bash
sudo suricata-update
sudo systemctl restart suricata
```

---

## 9. Detection Improvements

### Reduce Suricata False Positives

```bash
echo "suppress gen_id 1, sig_id 2210044, track by_src, ip <WAZUH_IP>" \
  | sudo tee -a /etc/suricata/threshold.conf

sudo systemctl restart suricata
```

### Wazuh Active Response — Auto-block Brute Force

Add to `/var/ossec/etc/ossec.conf` on the manager:

```xml
<active-response>
  <command>firewall-drop</command>
  <location>local</location>
  <rules_id>5763</rules_id>
  <timeout>300</timeout>
</active-response>
```

This automatically blocks the source IP of any SSH brute force for 5 minutes.

---

## 10. Troubleshooting

### Agent Cannot Connect to Manager

```bash
# Confirm correct manager IP
sudo grep -A3 "<server>" /var/ossec/etc/ossec.conf

# Test port reachability
nc -zv <WAZUH_IP> 1514
nc -zv <WAZUH_IP> 1515

# Check agent logs
sudo tail -f /var/ossec/logs/ossec.log
```

### Manager Ports Not Listening

```bash
sudo ss -tlnp | grep -E '1514|1515'
sudo systemctl restart wazuh-manager
```

### Logs Not Appearing in Dashboard

```bash
# Check indexer health
curl -k -u admin:<password> https://<WAZUH_IP>:9200/_cluster/health?pretty

# Restart all services
sudo systemctl restart wazuh-manager wazuh-indexer wazuh-dashboard filebeat
```

### No Traffic Captured in Suricata

```bash
# Confirm correct interface
grep "interface:" /etc/suricata/suricata.yaml

# Enable promiscuous mode
sudo ip link set <interface> promisc on

# Check logs
sudo journalctl -u suricata -f
sudo tail -f /var/log/suricata/eve.json
```

### Suricata Alerts Not Appearing in Wazuh

```bash
# Confirm eve.json is being written
sudo tail -5 /var/log/suricata/eve.json

# Check permissions
ls -la /var/log/suricata/eve.json
sudo chmod 644 /var/log/suricata/eve.json

# Confirm localfile entry in ossec.conf
sudo grep -A3 "suricata" /var/ossec/etc/ossec.conf

sudo systemctl restart wazuh-manager
```

### Dashboard Health Check Failing

```bash
sudo systemctl restart wazuh-manager wazuh-indexer wazuh-dashboard filebeat
sudo journalctl -u wazuh-dashboard -f
```

---

*Setup Guide v1.0 — SOC Detection Lab*
*See also: README.md | siem_lab_runbook.md*
