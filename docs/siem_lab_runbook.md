# SIEM Lab — Operational Runbook
### Exact Commands · Expected Alerts · Verification Steps

**Document Type:** Technical Runbook  
**Version:** 2.0  
**Environment:** Wazuh 4.14.5 · Suricata 8.0.3 · VMware Host-Only `192.168.172.0/24`  
**Companion To:** `siem_methodology_executive.md` · `setup-guide.md`

---

## Placeholder Reference

| Placeholder | Description |
|---|---|
| `<WAZUH_IP>` | Wazuh SIEM VM IP |
| `<KALI_IP>` | Kali Linux attacker IP |
| `<METASPLOITABLE3_IP>` | Metasploitable3 Ubuntu target IP |
| `<WINDOWS_11_IP>` | Windows 11 target IP |
| `<WINSERVER_2008_IP>` | Windows Server 2008 target IP |

---

## How to Use This Runbook

Each test follows the same five-step cycle. Do not skip steps — a missed step means the detection is unverified, not absent.

```
STEP 1 — Generate the event       Run the exact command on the correct VM
STEP 2 — Confirm raw log          Verify the event exists at the source
STEP 3 — Confirm ingestion        Wazuh Dashboard → Discover → search
STEP 4 — Confirm rule match       Check rule ID · alert level · decoder
STEP 5 — Tune if missing          See troubleshooting note in each section
```

---

## Pre-Flight Checks

Run these before starting any test phase. Nothing fires if agents are not connected and logs are not flowing.

### Confirm All Agents Are Active

**On Wazuh Manager:**
```bash
sudo /var/ossec/bin/agent_control -la
```
Expected output — all agents show `Active`:
```
ID: 001, Name: kali-agent,                  IP: <KALI_IP>,               Active
ID: 002, Name: metasploitable3-ubuntu-agent, IP: <METASPLOITABLE3_IP>,    Active
ID: 003, Name: windows-11-agent,             IP: <WINDOWS_11_IP>,         Active
```

**On Wazuh Dashboard:**
```
Menu → Agents → verify all agents show green "Active" status
```

### Confirm Suricata Is Running and Writing Alerts

```bash
sudo systemctl status suricata
sudo tail -5 /var/log/suricata/eve.json
```

Generate a quick test ping from Kali — events should appear within seconds:
```bash
# On Kali
ping -c 3 <WAZUH_IP>
# Back on Wazuh VM
sudo tail -f /var/log/suricata/eve.json
```

### Confirm Wazuh Is Ingesting Suricata Alerts

```bash
sudo grep -i "suricata\|eve.json" /var/ossec/logs/ossec.log | tail -10
```

---

---

# PHASE 1 — VISIBILITY
**Goal:** Confirm all log sources are flowing into Wazuh before running attack simulations.

---

## 1.1 — SSH Single Failed Login

**Run on Kali:**
```bash
ssh invaliduser@<METASPLOITABLE3_IP>
# Enter any wrong password when prompted
```

**Verify raw log on Metasploitable3:**
```bash
sudo tail -20 /var/log/auth.log
```
Expected line:
```
sshd[XXXX]: Failed password for invalid user invaliduser from <KALI_IP> port XXXXX ssh2
```

**Verify in Wazuh Dashboard:**
```
Discover → search: agent.name:"metasploitable3-ubuntu-agent" AND rule.id:5760
```

**Expected Alert:**
| Field | Value |
|---|---|
| Rule ID | 5760 |
| Level | 5 |
| Description | sshd: Attempt to login using a non-existent user |

**Tune if missing:** Confirm `auth.log` is listed in the linux-agents group config at `/var/ossec/etc/shared/linux-agents/agent.conf`. Restart the agent: `sudo systemctl restart wazuh-agent`.

---

## 1.2 — RDP Failed Login (Windows)

**On Windows 11 — lock screen or Run > mstsc to localhost:**
```
Attempt to log in with wrong password 3 times
```

**Verify raw log on Windows:**
```
Event Viewer → Windows Logs → Security → filter Event ID 4625
```

**Verify in Wazuh Dashboard:**
```
Discover → search: agent.name:"windows-11-agent" AND rule.id:60122
```

**Expected Alert:**
| Field | Value |
|---|---|
| Rule ID | 60122 |
| Level | 5 |
| Description | Windows: User logon failure |
| Event ID | 4625 |

**Tune if missing:** Confirm Security event channel is in the windows-agents group config. Check agent is Active in Dashboard.

---

## 1.3 — FIM Baseline Confirmation

**Trigger a known FIM event on Metasploitable3:**
```bash
sudo touch /etc/lab_fim_test
```

**Verify in Wazuh Dashboard:**
```
Discover → search: agent.name:"metasploitable3-ubuntu-agent" AND rule.id:554
```

**Expected Alert:**
| Rule ID | Level | Description |
|---|---|---|
| 554 | 5 | File added to monitored directory |

**Clean up:**
```bash
sudo rm /etc/lab_fim_test
```

**Tune if missing:** Confirm `/etc` is in the `<syscheck>` block of the linux-agents group config with `realtime="yes"`. Run `sudo /var/ossec/bin/agent_control -r -a` to force a FIM rescan.

---

---

# PHASE 2 — ATTACK PATTERNS
**Goal:** Validate detection of brute force and reconnaissance behavior.

---

## 2.1 — SSH Brute Force (Hydra)

**Run on Kali:**
```bash
hydra -l root -P /usr/share/wordlists/metasploit/unix_passwords.txt \
  ssh://<METASPLOITABLE3_IP> -t 4 -V
```

> Use `unix_passwords.txt` (shorter) for testing. `rockyou.txt` will work but takes much longer.

**Verify raw log on Metasploitable3:**
```bash
sudo tail -50 /var/log/auth.log | grep "Failed password"
```
Expected: rapid sequence of `Failed password for root from <KALI_IP>`

**Verify in Wazuh Dashboard:**
```
Discover → search: agent.name:"metasploitable3-ubuntu-agent" AND rule.id:5763
```

**Expected Wazuh Alerts (watch for all three):**
| Rule ID | Level | Description |
|---|---|---|
| 5760 | 5 | SSH: Invalid user login attempt |
| 5763 | 10 | SSH: Brute force trying to get access |
| 100002 | 12 | Custom: SSH brute force — 6+ failures in 60s |

**Expected Suricata Alert (fast.log):**
```bash
sudo tail -f /var/log/suricata/fast.log
```
Expected line:
```
[**] [1:9000010:2] LOCAL SSH Brute Force Attempt [**] [Classification: Attempted Administrator Privilege Gain]
```

**Tune if missing (Wazuh):** Custom rule 100002 fires on parent SID 5716. Confirm rule is loaded: `sudo /var/ossec/bin/wazuh-control restart`. Then re-run Hydra.

**Tune if missing (Suricata):** Confirm `local.rules` is referenced in `suricata.yaml` under `rule-files`. Run `sudo suricata -T -c /etc/suricata/suricata.yaml -i ens37` to validate config.

---

## 2.2 — RDP Brute Force (Hydra)

**Run on Kali:**
```bash
hydra -l Administrator -P /usr/share/wordlists/metasploit/unix_passwords.txt \
  rdp://<WINDOWS_11_IP> -t 1 -V
```

> `-t 1` is required. RDP does not handle parallel connections well.

**Verify raw log on Windows:**
```
Event Viewer → Security → filter Event ID 4625
Look for: multiple failures from same source IP in rapid succession
```

**Expected Wazuh Alerts:**
| Rule ID | Level | Description |
|---|---|---|
| 60122 | 5 | Windows: User logon failure |
| 60204 | 10 | Windows: Multiple logon failures — possible brute force |

**Expected Suricata Alert:**
```
[**] [1:9000050:1] LOCAL RDP Brute Force Attempt [**]
```

---

## 2.3 — Network Port Scan (Nmap)

**Run on Kali — three scan types:**
```bash
# 1. SYN scan — stealth, most common
nmap -sS <METASPLOITABLE3_IP>

# 2. Aggressive scan — OS detection, version, scripts
nmap -A <METASPLOITABLE3_IP>

# 3. Windows target
nmap -sS <WINDOWS_11_IP>
```

**Verify Suricata caught it:**
```bash
sudo grep "SCAN\|scan" /var/log/suricata/fast.log | tail -20
```

Expected lines:
```
[**] [1:9000001:1] ET SCAN Nmap SYN Scan [**]
[**] [1:9000002:1] ET SCAN Nmap ICMP Ping Sweep [**]
[**] [1:9000003:1] ET SCAN Nmap OS Detection [**]
```

**Verify in Wazuh Dashboard:**
```
Discover → search: rule.id:100032 OR rule.groups:"recon"
```

**Expected Wazuh Alert (via Suricata enrichment rule):**
| Rule ID | Level | Description |
|---|---|---|
| 100032 | 8 | Suricata: Network scan detected |

---

## 2.4 — Web Vulnerability Scan (Nikto)

**Run on Kali:**
```bash
nikto -h http://<METASPLOITABLE3_IP>
```

**Verify raw log on Metasploitable3:**
```bash
sudo tail -100 /var/log/apache2/access.log | grep -i nikto
```
Expected: flood of GET requests with `Nikto` in the User-Agent field.

**Verify Suricata caught it:**
```bash
sudo grep "Nikto" /var/log/suricata/fast.log
```
Expected:
```
[**] [1:9000030:1] LOCAL Nikto Web Scanner Detected [**]
```

**Expected Wazuh Alert:**
| Rule ID | Level | Description |
|---|---|---|
| 100010 | 10 | Nikto web vulnerability scanner detected in web logs |
| 31151 | 6 | Web scan detected (built-in Wazuh rule) |

---

---

# PHASE 3 — SYSTEM COMPROMISE SIMULATION
**Goal:** Validate endpoint detection — account manipulation, privilege escalation, file integrity.

---

## 3.1 — Create a New Local User (Windows)

**Run on Windows 11 (cmd as Administrator):**
```cmd
net user labattacker Password123! /add
```

**Verify raw log:**
```
Event Viewer → Security → Event ID 4720
```

**Verify in Wazuh Dashboard:**
```
Discover → search: agent.name:"windows-11-agent" AND rule.id:60145
```

**Expected Alert:**
| Rule ID | Level | Description | Event ID |
|---|---|---|---|
| 60145 | 8 | Windows: New user account created | 4720 |

**Clean up:**
```cmd
net user labattacker /delete
```

---

## 3.2 — Add User to Administrators Group (Windows)

**Run on Windows 11 (cmd as Administrator):**
```cmd
net user labattacker Password123! /add
net localgroup Administrators labattacker /add
```

**Verify raw log:**
```
Event Viewer → Security → Event ID 4732
```

**Expected Alerts:**
| Rule ID | Level | Description | Event ID |
|---|---|---|---|
| 60145 | 8 | Windows: New user account created | 4720 |
| 60146 | 8 | Windows: User added to privileged group | 4732 |

**Clean up:**
```cmd
net localgroup Administrators labattacker /delete
net user labattacker /delete
```

---

## 3.3 — Successful Login After Failures (Windows)

**Sequence on Windows 11:**
```
1. Lock screen (Win + L)
2. Enter wrong password 3 times
3. Enter correct password and log in successfully
```

**Verify raw log:**
```
Event Viewer → Security
Look for: 4625 entries (failures) immediately followed by 4624 (success)
```

**Expected Alert:**
| Rule ID | Level | Description |
|---|---|---|
| 60109 | 10 | Windows: Successful login after multiple failures |

---

## 3.4 — Sudo Privilege Escalation (Linux)

**Run on Metasploitable3:**
```bash
# Successful sudo escalation
sudo su

# Failed sudo — wrong password (repeat 3 times)
sudo su
# Enter wrong password each time
```

**Verify raw log:**
```bash
sudo tail -20 /var/log/auth.log
```
Expected lines:
```
sudo:   root : TTY=pts/0 ; PWD=/home/... ; USER=root ; COMMAND=/bin/su
sudo: pam_unix(sudo:auth): authentication failure
```

**Expected Wazuh Alerts:**
| Rule ID | Level | Description |
|---|---|---|
| 5401 | 5 | Sudo: Failed authentication attempt |
| 5402 | 5 | Sudo: Failed to run command |
| 5403 | 3 | Sudo: Successful privilege escalation |
| 100040 | 10 | Custom: Repeated sudo failures — 3+ in 60 seconds |

---

## 3.5 — File Integrity Monitoring — Linux

**Run on Metasploitable3:**
```bash
# Create a test file in a monitored directory
sudo touch /etc/lab_testfile

# Modify a safe monitored file
sudo bash -c 'echo "# lab test" >> /etc/hosts'

# Delete the test file
sudo rm /etc/lab_testfile
```

**Verify in Wazuh Dashboard:**
```
Discover → search: agent.name:"metasploitable3-ubuntu-agent" AND (rule.id:550 OR rule.id:553 OR rule.id:554)
```

**Expected Alerts:**
| Rule ID | Level | Description |
|---|---|---|
| 554 | 5 | File added to monitored directory |
| 550 | 7 | Integrity checksum changed |
| 553 | 7 | File deleted from monitored directory |

**Revert the hosts modification:**
```bash
sudo sed -i '/# lab test/d' /etc/hosts
```

---

## 3.6 — File Integrity Monitoring — Windows

**Run on Windows 11 (PowerShell as Administrator):**
```powershell
# Create
New-Item -Path "C:\Windows\Temp\labtest.txt" -ItemType File

# Modify
Add-Content -Path "C:\Windows\Temp\labtest.txt" -Value "lab modification"

# Delete
Remove-Item -Path "C:\Windows\Temp\labtest.txt"
```

**Expected Alerts:**
| Rule ID | Level | Description |
|---|---|---|
| 554 | 5 | Windows: New file in monitored path |
| 550 | 7 | Windows: File integrity change detected |
| 553 | 7 | Windows: File deleted from monitored path |

---

## 3.7 — FIM — Critical File: /etc/sudoers (Linux)

> **Caution:** This modifies a sensitive file. Use the test line shown — it is a comment only.

**Run on Metasploitable3:**
```bash
sudo bash -c 'echo "# lab_sudoers_test" >> /etc/sudoers'
```

**Expected Wazuh Alert (custom rule):**
| Rule ID | Level | Description |
|---|---|---|
| 100060 | 14 | FIM: Critical file modified — /etc/sudoers changed |

**Revert immediately:**
```bash
sudo sed -i '/# lab_sudoers_test/d' /etc/sudoers
sudo visudo -c   # verify sudoers is still valid
```

---

---

# PHASE 4 — ADVANCED BEHAVIOR
**Goal:** Exploitation simulation, persistence mechanisms, behavioral detection.

---

## 4.1 — vsftpd 2.3.4 Backdoor Exploit (Metasploitable3)

**Run on Kali:**
```bash
msfconsole -q

use exploit/unix/ftp/vsftpd_234_backdoor
set RHOSTS <METASPLOITABLE3_IP>
set RPORT 21
run
```
Expected: shell access established on port 6200.

**Verify Suricata caught the backdoor trigger:**
```bash
sudo grep "vsftpd\|6200" /var/log/suricata/fast.log
```
Expected:
```
[**] [1:9000040:1] LOCAL vsftpd 2.3.4 Backdoor Trigger Detected [**]
[**] [1:9000041:1] LOCAL vsftpd Backdoor Shell Connection — Port 6200 [**]
```

**Verify raw log on Metasploitable3:**
```bash
sudo tail -20 /var/log/vsftpd.log
sudo tail -20 /var/log/syslog
```

**Expected Wazuh Alerts:**
| Rule ID | Level | Description |
|---|---|---|
| 5712 | 10 | FTP anomalous authentication |
| 100031 | 10 | Custom: Suricata FTP anomaly — exploit attempt |

---

## 4.2 — Netcat Reverse Shell Simulation

**On Metasploitable3 — start listener:**
```bash
nc -lvp 4444
```

**On Kali — connect back:**
```bash
nc <METASPLOITABLE3_IP> 4444
```

**Verify Suricata caught the outbound connection:**
```bash
sudo grep "Reverse Shell\|4444" /var/log/suricata/fast.log
```
Expected:
```
[**] [1:9000042:1] LOCAL Possible Reverse Shell — Outbound Connection on Common Listener Port [**]
```

**Expected Wazuh Alerts:**
| Rule ID | Level | Description |
|---|---|---|
| 5710 | 7 | Suspicious network process detected |
| 92200 | 10 | Possible reverse shell / command execution |

**Clean up — kill listener on Metasploitable3:**
```bash
# Ctrl+C in the nc session
```

---

## 4.3 — Persistence via Cron (Linux)

**Run on Metasploitable3:**
```bash
# Add a non-destructive cron entry
(crontab -l 2>/dev/null; echo "* * * * * echo lab_persistence_test") | crontab -

# Verify it was written
crontab -l
```

**Verify raw log:**
```bash
sudo tail -20 /var/log/syslog | grep cron
```

**Expected Wazuh Alerts:**
| Rule ID | Level | Description |
|---|---|---|
| 5007 | 7 | cron: New crontab entry added |
| 100050 | 10 | Custom: New cron entry — possible persistence |

**Clean up:**
```bash
crontab -r
```

---

## 4.4 — Persistence via Scheduled Task (Windows)

**Run on Windows 11 (cmd as Administrator):**
```cmd
schtasks /create /tn "LabPersistenceTest" /tr "cmd.exe /c echo test" /sc minute /mo 1
```

**Verify raw log:**
```
Event Viewer → Security → Event ID 4698
```

**Expected Wazuh Alert:**
| Rule ID | Level | Description | Event ID |
|---|---|---|---|
| 60145 | 8 | Windows: Scheduled task created | 4698 |

**Clean up:**
```cmd
schtasks /delete /tn "LabPersistenceTest" /f
```

---

## 4.5 — Simulated Malware File Activity

**Run on Metasploitable3:**
```bash
# Mass file creation — ransomware-like behavior
mkdir /tmp/labtest && cd /tmp/labtest
for i in {1..100}; do echo "simulated content $i" > file$i.txt; done

# Base64 encoding — payload obfuscation simulation
echo "simulated_encoded_payload" | base64
base64 /etc/passwd > /tmp/labtest/encoded_out.txt
```

**Expected Wazuh Alerts (FIM must cover /tmp or auditd active):**
| Rule ID | Level | Description |
|---|---|---|
| 550 | 7 | Mass file creation — FIM spike |
| 92200 | 10 | Suspicious encoding command detected |

**Clean up:**
```bash
rm -rf /tmp/labtest
```

---

---

# QUICK REFERENCE — Rule ID Summary

## Wazuh Rules

| Rule ID | Level | Category | Description |
|---|---|---|---|
| 5760 | 5 | Auth | SSH: Invalid user login attempt |
| 5763 | 10 | Auth | SSH: Brute force detected |
| 5764 | 10 | Auth | SSH: Possible breakin attempt |
| 5401 | 5 | Privilege | Sudo: Failed authentication |
| 5402 | 5 | Privilege | Sudo: Failed to run command |
| 5403 | 3 | Privilege | Sudo: Successful escalation |
| 5007 | 7 | Persistence | cron: New crontab entry |
| 5710 | 7 | Exploit | Suspicious network process |
| 5712 | 10 | Exploit | FTP anomalous auth / exploit |
| 550 | 7 | FIM | File integrity change |
| 553 | 7 | FIM | File deleted from monitored path |
| 554 | 5 | FIM | New file added to monitored path |
| 60106 | 5 | Windows | Logon failure |
| 60109 | 10 | Windows | Success after multiple failures |
| 60122 | 5 | Windows | RDP / network logon failure |
| 60145 | 8 | Windows | User created / task created |
| 60146 | 8 | Windows | User added to privileged group |
| 60204 | 10 | Windows | Multiple logon failures — brute force |
| 92200 | 10 | Behavior | Suspicious command / encoding |

## Custom Rules (local_rules.xml)

| Rule ID | Level | Category | Description |
|---|---|---|---|
| 100001 | 10 | Recon | Nmap HTTP scan in web logs |
| 100002 | 12 | Auth | SSH brute force — 6+ failures in 60s |
| 100003 | 10 | Auth | SSH invalid username enumeration |
| 100010 | 10 | Recon | Nikto scanner in web logs |
| 100020 | 14 | Attack | Command injection in web logs |
| 100021 | 12 | Attack | SQL injection in web logs |
| 100022 | 12 | Attack | SQLMap scanner in web logs |
| 100030 | 8 | IDS | Suricata: SSH alert enrichment |
| 100031 | 10 | IDS | Suricata: FTP anomaly enrichment |
| 100032 | 8 | IDS | Suricata: Network scan enrichment |
| 100040 | 10 | Privilege | Repeated sudo failures |
| 100050 | 10 | Persistence | New cron entry added |
| 100060 | 14 | FIM | /etc/sudoers modified |
| 100061 | 12 | FIM | /etc/passwd or /etc/shadow modified |

## Suricata Rules (local.rules)

| SID | Category | Description |
|---|---|---|
| 9000001 | Recon | Nmap SYN scan |
| 9000002 | Recon | Nmap ICMP ping sweep |
| 9000003 | Recon | Nmap OS detection probe |
| 9000004 | Recon | Generic port sweep |
| 9000005 | Recon | Nmap HTTP scripting engine |
| 9000010 | Auth | SSH brute force threshold |
| 9000011 | Auth | Hydra SSH tool detected |
| 9000020 | Auth | FTP brute force threshold |
| 9000030 | Web | Nikto scanner |
| 9000031 | Web | SQLMap scanner |
| 9000032 | Web | SQLi — SELECT FROM |
| 9000033 | Web | SQLi — UNION SELECT |
| 9000034 | Web | Command injection in URI |
| 9000035 | Web | /etc/passwd via web |
| 9000040 | Exploit | vsftpd backdoor trigger |
| 9000041 | Exploit | vsftpd backdoor shell port 6200 |
| 9000042 | Exploit | Reverse shell outbound |
| 9000043 | Exploit | Metasploit stager PE download |
| 9000050 | Windows | RDP brute force |
| 9000051 | Windows | SMB port scan |

---

# MITRE ATT&CK Coverage Map

| Phase | Tactic | Technique ID | Technique |
|---|---|---|---|
| 2.1 · 2.2 | Credential Access | T1110 | Brute Force |
| 2.3 | Discovery | T1046 | Network Service Scanning |
| 2.3 | Reconnaissance | T1595 | Active Scanning |
| 2.4 | Discovery | T1595.002 | Vulnerability Scanning |
| 3.1 | Persistence | T1136 | Create Account |
| 3.2 | Privilege Escalation | T1548 | Abuse Elevation Control |
| 3.4 | Privilege Escalation | T1548.003 | Sudo Abuse |
| 3.5 · 3.6 | Defense Evasion | T1070 | Indicator Removal |
| 3.7 | Privilege Escalation | T1548.003 | Sudoers Modification |
| 4.1 | Initial Access | T1190 | Exploit Public-Facing Application |
| 4.2 | Execution | T1059 | Command & Scripting Interpreter |
| 4.3 | Persistence | T1053.003 | Scheduled Task: Cron |
| 4.4 | Persistence | T1053.005 | Scheduled Task: Windows |
| 4.5 | Defense Evasion | T1027 | Obfuscated Files / Encoding |

---

*Runbook v2.0 — SOC Detection Lab*  
*See also: `siem_methodology_executive.md` · `setup-guide.md` · `detection-improvements.md`*
