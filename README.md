# 🛡️ SOC Detection Lab — Wazuh + Suricata + Kali Linux

> A virtualized cybersecurity home lab simulating a SOC environment with SIEM deployment, network intrusion detection, host-based monitoring, and attack simulation. Built for practical detection engineering and portfolio demonstration.

---

## 📋 Table of Contents

1. [Lab Overview](#lab-overview)
2. [Architecture](#architecture)
3. [Repo Structure](#repo-structure)
4. [Environment Setup](#environment-setup)
   - [Host Requirements](#host-requirements)
   - [VM Configuration](#vm-configuration)
   - [Network Setup](#network-setup)
5. [Component Installation](#component-installation)
   - [Wazuh All-in-One Deployment](#wazuh-all-in-one-deployment)
   - [Suricata (Network IDS)](#suricata-network-ids)
   - [Suricata → Wazuh Integration](#suricata--wazuh-integration)
   - [Wazuh Agent — Metasploitable2](#wazuh-agent--metasploitable2)
   - [Wazuh Agent — Windows 11](#wazuh-agent--windows-11)
6. [Log Collection Configuration](#log-collection-configuration)
   - [SSH Authentication Logs](#ssh-authentication-logs)
   - [FTP Logs](#ftp-logs)
   - [Apache / DVWA Logs](#apache--dvwa-logs)
   - [Windows Event Logs](#windows-event-logs)
   - [Suricata Alerts](#suricata-alerts)
7. [Attack Scenarios](#attack-scenarios)
   - [Nmap Port Scan](#nmap-port-scan)
   - [SSH Brute Force (Hydra)](#ssh-brute-force-hydra)
   - [FTP Brute Force](#ftp-brute-force)
   - [DVWA Attacks](#dvwa-attacks)
8. [Detection Results](#detection-results)
9. [Custom Detection Rules](#custom-detection-rules)
10. [Troubleshooting](#troubleshooting)
11. [Detection Improvements](#detection-improvements)
12. [Screenshots](#screenshots)
13. [References](#references)

---

## Lab Overview

This lab demonstrates a realistic SOC (Security Operations Center) environment using open-source tooling. It combines:

| Capability | Tool |
|---|---|
| SIEM | Wazuh (Manager + Indexer + Dashboard) |
| Network IDS | Suricata |
| Host Monitoring | Wazuh Agents (Linux + Windows) |
| Attack Simulation | Kali Linux |
| Vulnerable Targets | Metasploitable2 (DVWA), Windows 11 |

**Key objectives:**
- Detect network-level attacks (Suricata: port scans, exploit signatures)
- Detect host-level attacks (Wazuh: brute force, log anomalies)
- Correlate network + host alerts in a single dashboard
- Simulate realistic attacker TTPs (MITRE ATT&CK aligned)

---

## Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                        HOST MACHINE                             │
│                      Ubuntu 24.04 LTS                           │
│                    (VMware Hypervisor)                          │
└──────────────────────────┬──────────────────────────────────────┘
                           │  Host-Only Network
           ┌───────────────┼───────────────────────┐
           │               │                       │
    ┌──────▼──────┐  ┌──────▼──────┐       ┌───────▼──────┐
    │  ATTACKER   │  │  SIEM / IDS │       │   TARGETS    │
    │             │  │             │       │              │
    │ Kali Linux  │  │   Wazuh     │       │Metasploitable│
    │             │  │  (Manager   │       │  2 (Linux)   │
    │ Tools:      │  │  +Indexer   │       │  DVWA/Apache │
    │ - Nmap      │  │  +Dashboard)│       │  FTP / SSH   │
    │ - Hydra     │  │             │       │              │
    │ - SQLMap    │  │  Suricata   │       ├──────────────┤
    │ - DVWA      │  │  (Network   │       │  Windows 11  │
    │   exploits  │  │   IDS)      │       │  (Event Logs)│
    │             │  │             │       │              │
    │192.168.56.x │  │192.168.56.10│       │192.168.56.20 │
    └─────────────┘  └─────────────┘       │192.168.56.30 │
                           │               └──────────────┘
                     ┌─────▼──────┐
                     │  Wazuh     │
                     │ Dashboard  │
                     │:5601 (Web) │
                     └────────────┘

Data Flows:
  Agents ──► Wazuh Manager (1514/TCP)
  Suricata eve.json ──► Wazuh (local file monitoring)
  Wazuh Manager ──► OpenSearch Indexer (9200/TCP)
  Browser ──► Wazuh Dashboard (443/HTTPS)
```

---

---

## Repo Structure

```
soc-detection-lab/
│
├── README.md
├── architecture/
│   └── lab-diagram.png
│
├── screenshots/
│   ├── wazuh/
│   │   ├── dashboard-overview.png
│   │   ├── agent-inventory.png
│   │   └── alerts-view.png
│   │
│   ├── suricata/
│   │   ├── eve-json-logs.png
│   │   └── alert-detection.png
│   │
│   ├── attacks/
│   │   ├── nmap-scan.png
│   │   ├── ssh-bruteforce.png
│   │   └── sqli-attack.png
│   │
│   └── windows/
│       └── event-logs.png
│
├── configs/
│   ├── wazuh/
│   │   ├── ossec.conf
│   │   └── local_rules.xml
│   │
│   ├── suricata/
│   │   ├── suricata.yaml
│   │   └── local.rules
│   │
│   └── agents/
│       ├── linux-agent.conf
│       └── windows-agent.conf
│
├── detection-rules/
│   ├── ssh-bruteforce.xml
│   ├── nmap-detection.xml
│   └── web-attacks.xml
│
├── attack-scenarios/
│   ├── nmap.md
│   ├── ssh-bruteforce.md
│   └── dvwa-sqli.md
│
└── docs/
    ├── setup-guide.md
    ├── troubleshooting.md
    └── detection-improvements.md

```

---

## Environment Setup

### Host Requirements

| Component | Minimum | Recommended |
|---|---|---|
| RAM | 16 GB | 32 GB |
| CPU | 4 cores | 8 cores |
| Disk | 100 GB | 200 GB |
| OS | Ubuntu 24.04 | Ubuntu 24.04 |

### VM Configuration

| VM | OS | RAM | Disk | Role |
|---|---|---|---|---|
| `wazuh-siem` | Ubuntu 22.04 LTS | 6 GB | 50 GB | Wazuh + Suricata |
| `kali-attacker` | Kali Linux (latest) | 2 GB | 30 GB | Attacker |
| `metasploitable2` | Metasploitable2 | 1 GB | 10 GB | Target (Linux) |
| `win11-target` | Windows 11 | 4 GB | 40 GB | Target (Windows) |

### Network Setup

All VMs are connected to a **Host-Only Network** (e.g., `192.168.56.0/24`) for isolation plus internet access via NAT adapter.

| VM | IP Address |
|---|---|
| Wazuh/Suricata | `192.168.56.10` |
| Kali Linux | `192.168.56.5` |
| Metasploitable2 | `192.168.56.20` |
| Windows 11 | `192.168.56.30` |

**VirtualBox setup:**
```bash
# Create Host-Only Network in VirtualBox
# File → Host Network Manager → Create
# Set: 192.168.56.1 / 255.255.255.0
# DHCP: Disabled (use static IPs)
```

Each VM needs **two network adapters**:
- Adapter 1: NAT (for internet/package downloads)
- Adapter 2: Host-Only (`vboxnet0`) (for lab traffic)

---

## Component Installation

### Wazuh All-in-One Deployment

Wazuh is deployed as an all-in-one instance (Manager + Indexer + Dashboard) on the `wazuh-siem` VM.

```bash
# Update system
sudo apt update && sudo apt upgrade -y

# Download and run Wazuh installer
curl -sO https://packages.wazuh.com/4.x/wazuh-install.sh
curl -sO https://packages.wazuh.com/4.x/config.yml

# Edit config.yml — set node IPs to your Wazuh VM IP
# nodes.indexer.ip: 192.168.56.10
# nodes.server.ip: 192.168.56.10
# nodes.dashboard.ip: 192.168.56.10

sudo bash wazuh-install.sh -a
```

> **Note:** Save the auto-generated admin credentials shown at the end of installation.

Access dashboard: `https://192.168.56.10` (default user: `admin`)

---

### Suricata (Network IDS)

Suricata is installed on the same `wazuh-siem` VM to capture lab network traffic.

```bash
# Install Suricata
sudo add-apt-repository ppa:oisf/suricata-stable -y
sudo apt update
sudo apt install suricata -y

# Check version
suricata --build-info | grep Version

# Update rules (Emerging Threats)
sudo suricata-update

# Identify the correct monitoring interface
ip a  # Look for your host-only interface (e.g., enp0s8)

# Edit Suricata config
sudo nano /etc/suricata/suricata.yaml
```

Key `suricata.yaml` settings:
```yaml
# Set home network
vars:
  address-groups:
    HOME_NET: "[192.168.56.0/24]"

# Set monitoring interface
af-packet:
  - interface: enp0s8   # Replace with your host-only adapter name

# Enable eve.json output
outputs:
  - eve-log:
      enabled: yes
      filetype: regular
      filename: /var/log/suricata/eve.json
      types:
        - alert
        - dns
        - http
        - ssh
        - flow
```

```bash
# Start and enable Suricata
sudo systemctl enable suricata
sudo systemctl start suricata
sudo systemctl status suricata

# Test traffic capture
sudo tail -f /var/log/suricata/eve.json
```

---

### Suricata → Wazuh Integration

Wazuh reads Suricata's `eve.json` via its local agent file monitoring capability.

```bash
# Edit Wazuh agent/manager local config
sudo nano /var/ossec/etc/ossec.conf
```

Add inside `<ossec_config>`:
```xml
<localfile>
  <log_format>json</log_format>
  <location>/var/log/suricata/eve.json</location>
</ossec_config>
```

```bash
# Restart Wazuh manager
sudo systemctl restart wazuh-manager

# Verify logs are being read
sudo tail -f /var/ossec/logs/ossec.log | grep suricata
```

---

### Wazuh Agent — Metasploitable2

```bash
# On Metasploitable2 VM
# Download agent (adjust version to match your Wazuh server)
wget https://packages.wazuh.com/4.x/apt/pool/main/w/wazuh-agent/wazuh-agent_4.x.x_amd64.deb

# Install with manager IP
sudo WAZUH_MANAGER='192.168.56.10' dpkg -i wazuh-agent_4.x.x_amd64.deb

# Start agent
sudo systemctl daemon-reload
sudo systemctl enable wazuh-agent
sudo systemctl start wazuh-agent
```

---

### Wazuh Agent — Windows 11

1. Download the Wazuh Windows agent MSI from: `https://packages.wazuh.com/4.x/windows/`
2. Run installer and set Manager IP: `192.168.56.10`
3. Or install via PowerShell:

```powershell
# Run as Administrator
Invoke-WebRequest -Uri "https://packages.wazuh.com/4.x/windows/wazuh-agent-4.x.x-1.msi" -OutFile "wazuh-agent.msi"
msiexec.exe /i wazuh-agent.msi /q WAZUH_MANAGER="192.168.56.10" WAZUH_AGENT_NAME="win11-target"

# Start service
NET START WazuhSvc
```

---

## Log Collection Configuration

### SSH Authentication Logs

Wazuh monitors `/var/log/auth.log` (Debian/Ubuntu) by default. On Metasploitable2:

```xml
<!-- In ossec.conf on the Metasploitable2 agent -->
<localfile>
  <log_format>syslog</log_format>
  <location>/var/log/auth.log</location>
</localfile>
```

### FTP Logs

```xml
<localfile>
  <log_format>syslog</log_format>
  <location>/var/log/vsftpd.log</location>
</localfile>
```

### Apache / DVWA Logs

```xml
<localfile>
  <log_format>apache</log_format>
  <location>/var/log/apache2/access.log</location>
</localfile>

<localfile>
  <log_format>apache</log_format>
  <location>/var/log/apache2/error.log</location>
</localfile>
```

### Windows Event Logs

Windows agent `ossec.conf` (at `C:\Program Files (x86)\ossec-agent\ossec.conf`):

```xml
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
  <location>Application</location>
</localfile>
```

### Suricata Alerts

Already configured above via `eve.json` ingestion into Wazuh manager's `ossec.conf`.

---

## Attack Scenarios

### Nmap Port Scan

**From Kali Linux:**
```bash
# SYN scan (stealthy)
sudo nmap -sS -p- 192.168.56.20

# Service version + OS detection
sudo nmap -sV -O 192.168.56.20

# Aggressive scan
sudo nmap -A 192.168.56.20
```

**Expected detection:**
- Suricata: `ET SCAN` rules trigger on port sweep
- Wazuh Rule: `40101` — Port scan detected

---

### SSH Brute Force (Hydra)

```bash
# Generate or use wordlist
hydra -l root -P /usr/share/wordlists/rockyou.txt \
  ssh://192.168.56.20 -t 4 -V
```

**Expected detection:**
- Wazuh Rule: `5712` — SSH brute force (multiple failed logins)
- Auth log: Multiple `Failed password` entries

---

### FTP Brute Force

```bash
hydra -l ftp -P /usr/share/wordlists/rockyou.txt \
  ftp://192.168.56.20 -V
```

**Expected detection:**
- Wazuh Rule: `11201` — FTP authentication failure
- Suricata: `ET SCAN FTP Brute Force` signature

---

### DVWA Attacks

Ensure DVWA is accessible at `http://192.168.56.20/dvwa`

**SQL Injection:**
```bash
sqlmap -u "http://192.168.56.20/dvwa/vulnerabilities/sqli/?id=1&Submit=Submit" \
  --cookie="PHPSESSID=<your_session>; security=low" \
  --dbs --batch
```

**Command Injection (manual):**
```
# In DVWA Command Injection page:
127.0.0.1; cat /etc/passwd
127.0.0.1 && id
```

**Brute Force (DVWA login):**
```bash
hydra -l admin -P /usr/share/wordlists/rockyou.txt \
  192.168.56.20 http-get-form \
  "/dvwa/vulnerabilities/brute/:username=^USER^&password=^PASS^&Login=Login:Username and/or password incorrect.:H=Cookie: PHPSESSID=<session>; security=low"
```

**Expected detection:**
- Apache access log: Suspicious GET/POST patterns
- Wazuh Rule: `31103` — Web attack SQL injection attempt
- Suricata: `ET WEB_SERVER` rules

---

## Detection Results

| Attack | Detection Source | Wazuh Rule ID | Alert Level |
|---|---|---|---|
| Nmap SYN Scan | Suricata | 40101 | Medium |
| SSH Brute Force | Auth logs (Wazuh) | 5712 | High |
| FTP Brute Force | vsftpd logs (Wazuh) | 11201 | High |
| SQL Injection | Apache logs (Wazuh) | 31103 | High |
| Command Injection | Apache logs (Wazuh) | 31106 | Critical |
| DVWA Brute Force | Apache logs (Wazuh) | 31151 | High |

> ⚠️ **Note:** Rule IDs may vary depending on Wazuh version. Verify in your deployment.

---

## Custom Detection Rules

Custom rules are placed at `/var/ossec/etc/rules/local_rules.xml`.

**Example — Detect Nmap User-Agent in HTTP:**
```xml
<group name="web,nmap,">
  <rule id="100001" level="10">
    <if_group>web|accesslog</if_group>
    <match>Nmap Scripting Engine</match>
    <description>Nmap HTTP scan detected in web logs</description>
    <mitre>
      <id>T1046</id>
    </mitre>
  </rule>
</group>
```

**Example — Multiple Failed SSH Logins (custom threshold):**
```xml
<group name="syslog,ssh,">
  <rule id="100002" level="12" frequency="6" timeframe="60">
    <if_matched_sid>5716</if_matched_sid>
    <description>SSH brute force: 6+ failed attempts in 60 seconds</description>
    <mitre>
      <id>T1110</id>
    </mitre>
  </rule>
</group>
```

```bash
# Reload rules without restarting
sudo /var/ossec/bin/wazuh-control restart
```

---

## Troubleshooting

### No Traffic Captured in Suricata

```bash
# Verify interface name
ip link show

# Test Suricata is capturing on the right interface
sudo suricata -i enp0s8 --pidfile /tmp/suricata.pid

# Check for errors
sudo journalctl -u suricata -f

# Verify promiscuous mode (if needed)
sudo ip link set enp0s8 promisc on
```

### Agents Not Reporting to Wazuh

```bash
# On agent machine — check connectivity
telnet 192.168.56.10 1514

# Check agent status
sudo systemctl status wazuh-agent

# View agent logs
sudo tail -f /var/ossec/logs/ossec.log

# On Wazuh manager — list agents
sudo /var/ossec/bin/agent_control -la
```

### Logs Not Appearing in Dashboard

```bash
# Check indexer health
curl -k -u admin:<password> https://192.168.56.10:9200/_cluster/health?pretty

# Check Wazuh manager is indexing
sudo tail -f /var/ossec/logs/ossec.log | grep indexer

# Restart services
sudo systemctl restart wazuh-manager
sudo systemctl restart wazuh-indexer
sudo systemctl restart wazuh-dashboard
```

### Suricata Alerts Not in Wazuh

```bash
# Verify eve.json is being written
sudo tail -f /var/log/suricata/eve.json

# Check file permissions
ls -la /var/log/suricata/eve.json
sudo chmod 644 /var/log/suricata/eve.json

# Verify ossec.conf has the localfile entry
sudo grep -A3 "suricata" /var/ossec/etc/ossec.conf
```

---

## Detection Improvements

### Custom Suricata Rules

Place custom rules in `/etc/suricata/rules/local.rules`:

```bash
# Detect Hydra SSH brute force tool
alert tcp any any -> $HOME_NET 22 (msg:"HYDRA SSH Brute Force Tool Detected"; \
  flow:to_server,established; content:"SSH"; threshold:type threshold, \
  track by_src, count 5, seconds 60; sid:9000001; rev:1;)

# Detect DVWA SQLi patterns
alert http any any -> $HOME_NET any (msg:"DVWA SQLi Attempt"; \
  http.uri; content:"SELECT"; nocase; content:"FROM"; nocase; \
  sid:9000002; rev:1;)
```

After adding rules:
```bash
sudo suricata-update
sudo systemctl restart suricata
```

### Reducing False Positives

```bash
# Suppress noisy Suricata rules
echo "suppress gen_id 1, sig_id 2210044, track by_src, ip 192.168.56.10" \
  >> /etc/suricata/threshold.conf

# Tune Wazuh rule levels in local_rules.xml
# Reduce level or add exceptions for known-good traffic
```

### Wazuh Active Response

Enable automated blocking on brute force:
```xml
<!-- In ossec.conf -->
<active-response>
  <command>firewall-drop</command>
  <location>local</location>
  <rules_id>5712</rules_id>
  <timeout>300</timeout>
</active-response>
```

---

## Screenshots

> *Screenshots to be added as lab is built and attacks are executed.*

| Screenshot | Description |
|---|---|
| `screenshots/01_wazuh_dashboard.png` | Wazuh main dashboard overview |
| `screenshots/02_suricata_alerts.png` | Suricata network alerts in Wazuh |
| `screenshots/03_ssh_brute_force.png` | SSH brute force alert chain |
| `screenshots/04_nmap_scan_detection.png` | Nmap port scan detection |
| `screenshots/05_dvwa_sqli_alert.png` | SQL injection web attack alert |
| `screenshots/06_agent_inventory.png` | All agents connected in dashboard |
| `screenshots/07_windows_events.png` | Windows 11 event log ingestion |

---

## References

- [Wazuh Official Documentation](https://documentation.wazuh.com)
- [Suricata Documentation](https://suricata.readthedocs.io)
- [Emerging Threats Rules](https://rules.emergingthreats.net)
- [MITRE ATT&CK Framework](https://attack.mitre.org)
- [DVWA Project](https://dvwa.co.uk)

---

## Author

**Wilson Njoroge Wanderi**
Cybersecurity Detection Engineer — SOC Lab Portfolio Project

> Built as a hands-on demonstration of SIEM deployment, network intrusion detection, and attack simulation in a virtualized environment.

---

*README v1.0 — Updated as lab is built*
