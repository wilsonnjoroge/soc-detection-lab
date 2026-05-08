# Detection Improvements
### Tuning · Hardening · Coverage Gaps

**Document Type:** Operational Reference  
**Version:** 1.0  
**Environment:** Wazuh 4.14.5 · Suricata 8.0.3 · VMware Host-Only `192.168.172.0/24`  
**Purpose:** Documents known gaps, noise reduction steps, active response config, and future improvements

---

## 1. Suricata — Reducing False Positives

### Problem
During normal lab operation, certain Suricata rules fire on legitimate traffic — particularly internal scanning between VMs during setup, Wazuh agent health checks, and update traffic. This creates noise that buries real alerts.

### Fix 1 — Suppress Known Noisy Signatures

Add suppression entries to `/etc/suricata/threshold.conf`:

```bash
sudo nano /etc/suricata/threshold.conf
```

```
# Suppress Wazuh agent-to-manager heartbeat traffic
suppress gen_id 1, sig_id 2210044, track by_src, ip <WAZUH_IP>

# Suppress internal VM-to-VM ICMP if causing false ping sweep alerts
suppress gen_id 1, sig_id 9000002, track by_src, ip <WAZUH_IP>

# Suppress update traffic from known hosts
suppress gen_id 1, sig_id 9000004, track by_src, ip <KALI_IP>
```

Apply:
```bash
sudo systemctl restart suricata
```

### Fix 2 — Tune Thresholds for Lab Traffic Volume

The lab runs on a small network. Some thresholds may be too sensitive.  
Adjust the following in `local.rules` if rules fire on routine traffic:

| SID | Current Threshold | Suggested Adjustment |
|---|---|---|
| 9000004 (port sweep) | 20 SYNs / 5s | Raise to 30 / 5s if Nmap install scans trigger it |
| 9000002 (ICMP sweep) | 5 packets / 3s | Raise to 10 / 3s if routine pings cause noise |
| 9000010 (SSH brute) | 10 SYNs / 30s | Keep — already conservative |

After editing `local.rules`:
```bash
sudo suricata-update
sudo systemctl restart suricata
sudo suricata -T -c /etc/suricata/suricata.yaml -i ens37   # validate before restart
```

---

## 2. Wazuh — Active Response Configuration

Active response automatically blocks attacker IPs when specific rules fire. This is configured in `ossec.conf` on the manager.

### Currently Configured Responses

| Trigger Rule | Level | Action | Duration |
|---|---|---|---|
| 5763 — SSH brute force | 10 | `firewall-drop` on agent | 300 seconds |
| 60204 — Windows RDP brute force | 10 | `firewall-drop` on agent | 300 seconds |

### How to Verify Active Response Is Working

**After running Hydra SSH brute force (Phase 2.1), confirm the block was applied:**

On Metasploitable3:
```bash
sudo iptables -L -n | grep DROP
# Expected: DROP rule for <KALI_IP> added by ossec-hids
```

On the Wazuh manager, check the active response log:
```bash
sudo tail -50 /var/ossec/logs/active-responses.log
```
Expected:
```
active-response/bin/firewall-drop add - <KALI_IP> ...
```

### Unblock a Manually Stuck IP

If the block persists after the timeout, or you need to unblock for continued testing:
```bash
# On the agent where the block was applied
sudo iptables -D INPUT -s <KALI_IP> -j DROP
sudo iptables -D FORWARD -s <KALI_IP> -j DROP
```

Or force unblock from the manager:
```bash
sudo /var/ossec/bin/agent_control -b <KALI_IP> -f firewall-drop0 -u <AGENT_ID>
```

### Adding More Active Response Triggers

To block command injection or web attack IPs, add to `ossec.conf`:

```xml
<!-- Block IPs triggering command injection rule -->
<active-response>
  <command>firewall-drop</command>
  <location>defined-agent</location>
  <agent_id>002</agent_id>   <!-- metasploitable3-ubuntu-agent ID -->
  <rules_id>100020</rules_id>
  <timeout>600</timeout>
</active-response>
```

Restart manager after changes:
```bash
sudo systemctl restart wazuh-manager
```

---

## 3. Wazuh — Improving Alert Fidelity

### Problem: Custom Rules Not Firing

Custom rules in `local_rules.xml` depend on parent rules (`if_sid` / `if_matched_sid`). If the parent rule does not fire, the child never will.

**Diagnostic steps:**
```bash
# Test a raw log line against the Wazuh rule engine interactively
sudo /var/ossec/bin/wazuh-logtest

# Paste a raw log line and observe decoder + rule matches
# Example input line:
# Feb 28 10:15:33 metasploitable sshd[1234]: Failed password for root from 192.168.172.200 port 50000 ssh2
```

The output shows exactly which decoder matched and which rule fired. If your custom rule should have fired but didn't, verify the parent `if_matched_sid` value is correct.

### Problem: Rule Fires but Level Too Low to Appear

Wazuh Dashboard defaults to level 3 and above. If a rule fires at level 3 but you expect it at level 7, it may be filtered in the view.

```
Dashboard → Security Events → filter: rule.level >= 3
```

### Problem: FIM Alerts Delayed

Wazuh FIM uses real-time inotify for paths marked `realtime="yes"`, and periodic scan for everything else. Default scan interval is 300 seconds.

For testing, always use directories marked `realtime="yes"` — alerts will fire within seconds. For periodic-scan paths, force a rescan:

```bash
sudo /var/ossec/bin/agent_control -r -a       # all agents
sudo /var/ossec/bin/agent_control -r -u 002   # specific agent by ID
```

---

## 4. Suricata — Expanding Rule Coverage

### Add Emerging Threats Ruleset (Recommended)

The default Suricata install includes only basic rules. The Emerging Threats Open ruleset adds thousands of community-maintained signatures covering FTP exploits, web shells, and C2 patterns.

```bash
sudo suricata-update update-sources
sudo suricata-update enable-source et/open
sudo suricata-update
sudo systemctl restart suricata
```

Verify ET rules are loaded:
```bash
sudo suricata-update list-enabled-sources
```

### Add Protocol Anomaly Rules

Add the following to `local.rules` for additional coverage:

```
# FTP AUTH flood — generic FTP brute force
alert tcp any any -> $HOME_NET 21 (
  msg:"LOCAL FTP AUTH Flood";
  flow:to_server, established;
  content:"PASS";
  threshold:type threshold, track by_src, count 15, seconds 60;
  classtype:attempted-admin;
  sid:9000060;
  rev:1;
)

# Telnet connection — Metasploitable3 has telnet exposed
alert tcp any any -> $HOME_NET 23 (
  msg:"LOCAL Telnet Connection Attempt";
  flags:S;
  classtype:attempted-admin;
  sid:9000061;
  rev:1;
)
```

---

## 5. Windows — Sysmon Integration

### Why Sysmon Improves Detection

Windows Event Logs capture authentication events well but miss process creation, network connections, and file writes. Adding Sysmon unlocks:

| Event ID | What It Captures |
|---|---|
| 1 | Process creation — catches PowerShell, cmd.exe spawned by exploits |
| 3 | Network connection — catches reverse shells dialing out |
| 11 | File creation — complements Wazuh FIM |
| 13 | Registry modification — catches persistence |

### Install Sysmon on Windows 11

```powershell
# Run as Administrator
Invoke-WebRequest -Uri "https://download.sysinternals.com/files/Sysmon.zip" -OutFile "$env:TEMP\Sysmon.zip"
Expand-Archive "$env:TEMP\Sysmon.zip" -DestinationPath "$env:TEMP\Sysmon"

# Download SwiftOnSecurity baseline config
Invoke-WebRequest -Uri "https://raw.githubusercontent.com/SwiftOnSecurity/sysmon-config/master/sysmonconfig-export.xml" -OutFile "$env:TEMP\sysmonconfig.xml"

# Install
& "$env:TEMP\Sysmon\Sysmon64.exe" -accepteula -i "$env:TEMP\sysmonconfig.xml"
```

The windows-agents group config already includes the Sysmon channel collector — once Sysmon is installed, events flow into Wazuh automatically.

**Verify in Dashboard:**
```
Discover → search: agent.name:"windows-11-agent" AND data.win.system.channel:"Microsoft-Windows-Sysmon/Operational"
```

---

## 6. Known Coverage Gaps

| Gap | Technique | Reason | Planned Fix |
|---|---|---|---|
| PowerShell encoded commands | T1027 | Sysmon not yet installed | Install Sysmon — Event ID 1 captures encoded PS |
| Lateral movement via SMB | T1021.002 | SMB rules only cover port scan | Add Wazuh rule for Event ID 4624 logon type 3 |
| DNS tunneling | T1071.004 | DNS not enriched in Wazuh | Enable Suricata DNS logger + Wazuh DNS rules |
| DVWA login brute force | T1110 | HTTP POST body not inspected | Enable Suricata HTTP body inspection |
| Meterpreter encrypted C2 | T1573 | Encrypted — not decodable | Behavioral rules: connection frequency thresholds |
| Exfiltration via FTP | T1048 | FTP data channel not monitored | Add Suricata FTP data transfer threshold rules |

---

## 7. Dashboard Saved Searches (Recommended)

Create these in Wazuh Dashboard → Discover for fast daily triage:

| Search Name | Query |
|---|---|
| All High Alerts | `rule.level:>=10` |
| SSH Brute Force | `rule.id:(5763 OR 100002)` |
| Windows Auth Failures | `rule.id:(60122 OR 60204 OR 60109)` |
| All Suricata Alerts | `data.event_type:alert` |
| FIM Changes | `rule.groups:syscheck AND rule.level:>=7` |
| Web Attacks | `rule.id:(100020 OR 100021 OR 100022 OR 100010)` |
| Persistence | `rule.id:(5007 OR 100050 OR 60145)` |
| Custom Rules Only | `rule.id:>=100000` |

---

*Detection Improvements v1.0 — SOC Detection Lab*  
*See also: `siem_lab_runbook.md` · `local_rules.xml` · `local.rules` · `ossec.conf`*
