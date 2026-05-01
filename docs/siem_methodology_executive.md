# SIEM & SOC Lab Testing Methodology
### Security Operations Center — Detection Validation Framework

**Document Type:** Executive Methodology Overview
**Environment:** Wazuh SIEM | Kali Linux · Windows VM · Metasploitable2
**Standard:** MITRE ATT&CK Framework Aligned

---

## Overview

This document defines a structured, repeatable methodology for validating detection coverage across a Wazuh-based SOC lab environment. Rather than executing tools arbitrarily, this framework maps every test action to a telemetry domain, an expected log source, and a measurable detection outcome.

The goal is not to "run attacks" — it is to **prove that your SIEM sees what it should see**.

---

## Lab Environment

| Agent | Role | Primary Log Sources |
|---|---|---|
| Wazuh Server | Manager · Indexer · Dashboard | Wazuh alerts, rule engine |
| Kali Linux | Attacker simulation | auth.log, syslog, audit.log |
| Windows VM | Enterprise endpoint | Windows Event Log (Security, System) |
| Metasploitable2 | Vulnerable target | syslog, auth.log, service logs |

---

## Core Testing Philosophy

Detection coverage is measured across **five telemetry domains**. Every test maps to one of these:

| Domain | What It Covers |
|---|---|
| **Authentication Events** | Login failures, brute force, credential attacks |
| **Privilege & User Activity** | Account creation, group changes, sudo abuse |
| **Network Attack Patterns** | Reconnaissance, scanning, lateral movement |
| **System Changes** | File integrity, registry, configuration drift |
| **Malware & Exploit Behavior** | Exploitation, persistence, abnormal processes |

---

## Detection Coverage Matrix

### A. Authentication Attacks
**MITRE ATT&CK:** T1110 (Brute Force) · T1078 (Valid Accounts)
**Priority:** Critical

| Action | Log Source | Windows Event ID | Wazuh Rule ID |
|---|---|---|---|
| Failed SSH login | /var/log/auth.log | — | 5760 |
| SSH brute force (Hydra) | /var/log/auth.log | — | 5763 |
| Failed RDP login | Windows Security Log | 4625 | 60122 |
| Failed local login | Windows Security Log | 4625 | 60106 |
| Successful login after failures | Windows Security Log | 4624 | 60109 |

---

### B. Privilege & User Activity
**MITRE ATT&CK:** T1136 (Create Account) · T1078 (Valid Accounts) · T1548 (Privilege Escalation)
**Priority:** Critical

| Action | Log Source | Windows Event ID | Wazuh Rule ID |
|---|---|---|---|
| New local user created | Windows Security Log | 4720 | 60145 |
| User added to Admins group | Windows Security Log | 4732 | 60146 |
| Sudo privilege escalation | /var/log/auth.log | — | 5401 |
| Failed sudo attempt | /var/log/auth.log | — | 5402 |

---

### C. Network Reconnaissance
**MITRE ATT&CK:** T1046 (Network Service Scanning) · T1595 (Active Scanning)
**Priority:** High

| Action | Log Source | Detection Method | Wazuh Rule ID |
|---|---|---|---|
| Port scan (nmap -sS) | Network / IDS | Repeated SYN packets | 40101 |
| Aggressive scan (nmap -A) | Network / IDS | Multi-probe pattern | 40111 |
| Web vulnerability scan (Nikto) | Web server logs | High-frequency HTTP errors | 31151 |

---

### D. File Integrity Monitoring (FIM)
**MITRE ATT&CK:** T1565 (Data Manipulation) · T1070 (Indicator Removal)
**Priority:** High

| Action | Log Source | Detection | Wazuh Rule ID |
|---|---|---|---|
| Critical file created | Wazuh FIM | Inotify / checksum | 550 |
| Critical file modified | Wazuh FIM | Hash change | 550 |
| Critical file deleted | Wazuh FIM | Deletion event | 553 |
| Windows system file changed | Wazuh FIM (Windows) | Registry / file watch | 550 |

---

### E. Exploitation & Malware Behavior
**MITRE ATT&CK:** T1190 (Exploit Public-Facing Application) · T1059 (Command & Scripting)
**Priority:** High

| Action | Log Source | Detection | Wazuh Rule ID |
|---|---|---|---|
| vsftpd backdoor exploit | Service log / syslog | Anomalous connection | 5712 |
| Netcat listener (reverse shell) | syslog / audit.log | Suspicious process | 5710 |
| Mass file creation | Wazuh FIM | FIM spike | 550 |
| Encoded payload execution | audit.log | Suspicious command | 92200 |

---

### F. Persistence Techniques
**MITRE ATT&CK:** T1053 (Scheduled Task) · T1547 (Boot/Logon Autostart)
**Priority:** Medium

| Action | Log Source | Windows Event ID | Wazuh Rule ID |
|---|---|---|---|
| New cron job (Linux) | /var/log/syslog | — | 5007 |
| New scheduled task (Windows) | Windows Security Log | 4698 | 60145 |
| Startup registry modification | Windows Registry | — | 92200 |

---

## Validation Cycle

Every test must complete all five steps. Partial validation is not coverage.

```
STEP 1 — Generate the event
         Execute the action on the target VM

STEP 2 — Confirm raw log
         Verify the log exists at the source (auth.log, Event Viewer, syslog)

STEP 3 — Confirm ingestion
         Wazuh Dashboard → Discover → search for the agent + event

STEP 4 — Confirm rule match
         Check: alert level fired · rule ID triggered · decoder matched

STEP 5 — Tune if missing
         Enable log collector · adjust decoder · verify rule is active
```

---

## Recommended Testing Progression

| Phase | Focus | Objective |
|---|---|---|
| **Phase 1 — Visibility** | SSH failures, RDP failures, basic log flow | Confirm log ingestion from all agents |
| **Phase 2 — Attack Patterns** | Hydra brute force, nmap scans | Validate pattern-based detection |
| **Phase 3 — System Compromise** | User creation, privilege escalation, FIM | Validate endpoint detection depth |
| **Phase 4 — Advanced Behavior** | Metasploit exploits, persistence simulation | Validate behavioral/heuristic rules |

---

## Coverage Summary

By completing this methodology, the following MITRE ATT&CK tactic categories are exercised:

| Tactic | Techniques Covered |
|---|---|
| Initial Access | T1190, T1078 |
| Execution | T1059 |
| Persistence | T1053, T1547 |
| Privilege Escalation | T1548, T1078 |
| Defense Evasion | T1070 |
| Discovery | T1046, T1595 |
| Impact | T1565 |

---

## Key Principle

> The test is not complete when the attack runs.
> The test is complete when Wazuh generates a verifiable, rule-matched alert — and you can explain why.

---

*For exact commands, expected output, and per-phase runbook — refer to: **SIEM Lab Operational Runbook***
