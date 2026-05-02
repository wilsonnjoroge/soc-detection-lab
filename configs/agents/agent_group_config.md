# Wazuh Agent Deployment & Group-Based Configuration
### SOC Lab — Centralized Policy Enforcement

**Document Type:** Architecture & Configuration Reference
**Environment:** Wazuh Manager · Kali Linux · Windows VM · Metasploitable2
**Current State:** All agents on `default` group → migration to OS-specific groups pending

---

## Overview

This document covers how Wazuh agents are deployed and managed using group-based configuration in a SOC lab environment. The objective is to move away from the `default` group and implement a proper OS-aware policy model — the same pattern used in production SOC deployments.

**Goals:**
- Centralized agent management from the manager
- OS-specific configurations (Linux vs Windows)
- Consistent FIM and log collection across all endpoints
- Scalable, auditable, drift-free architecture

---

## Architecture

```
Agent (Linux / Windows)
        │
        │  Secure TCP :1514
        ▼
Wazuh Manager
        │
        ├── Rules + Decoders
        ├── Group Policy Distribution
        └── Alert Engine
                │
                ▼
        Dashboard (OpenSearch)
```

---

## Current State vs Target State

| | Current | Target |
|---|---|---|
| Linux agents | `default` group | `linux-agents` group |
| Windows agents | `default` group | `windows-agents` group |
| FIM config | Default (minimal) | OS-tuned, realtime on critical paths |
| Log collection | Default | Auth logs, syslog, Windows Event Log |
| Scalability | Manual per-agent | Group-push from manager |

> **Action required:** Migrate all agents from `default` to their respective OS groups. See [Migration Steps](#migration-steps) below.

---

## Agent Groups — Concept

Wazuh groups are logical policy containers. Instead of configuring each agent manually, you define a group once on the manager and push it to all matching agents.

```
/var/ossec/etc/shared/
├── default/
│   └── agent.conf          ← current (avoid relying on this long-term)
├── linux-agents/
│   └── agent.conf          ← Linux policy: FIM, auth logs, syslog
└── windows-agents/
    └── agent.conf          ← Windows policy: Event Log, registry, FIM
```

### Why groups matter in a SOC context

| Without Groups | With Groups |
|---|---|
| Manual config per agent | Centralized, manager-controlled |
| Configuration drift | Consistent policy enforcement |
| Hard to audit | Single source of truth per OS |
| Doesn't scale | Scalable to any number of endpoints |

---

## Configuration Enforcement Flow

```
1. Assign agent to group
         ↓
2. Manager reads group's agent.conf
         ↓
3. Manager pushes config to agent (on connect / restart / sync)
         ↓
4. Agent enforces policy locally
   (FIM rules, log paths, monitoring settings)
```

| Component | Role |
|---|---|
| `agent.conf` in group folder | Defines the policy |
| Wazuh Manager | Distributes policy to assigned agents |
| Wazuh Agent | Enforces policy locally |

> **Sync note:** Config changes are not instantaneous. Restarting the agent forces immediate sync. Organic sync happens on reconnect or periodic check-in.

---

## Migration Steps
### Moving agents from `default` → OS-specific groups

**Step 1 — Create the groups (on the manager):**
```bash
sudo /var/ossec/bin/agent_groups -a -g linux-agents
sudo /var/ossec/bin/agent_groups -a -g windows-agents

# or via API / Agent Management: Groups→ Add new group
```

**Step 2 — Find your agent IDs:**
```bash
sudo /var/ossec/bin/manage_agents -l
# or via API / Dashboard: Agents → list
```

**Step 3 — Assign agents to groups:**
```bash
# Replace 001, 002 etc. with your actual agent IDs
sudo /var/ossec/bin/agent_groups -a -i 001 -g linux-agents    # Kali
sudo /var/ossec/bin/agent_groups -a -i 002 -g linux-agents    # Metasploitable2
sudo /var/ossec/bin/agent_groups -a -i 003 -g windows-agents  # Windows VM
```

**Step 4 — Verify assignment:**
```bash
sudo /var/ossec/bin/agent_groups -l -g linux-agents
sudo /var/ossec/bin/agent_groups -l -g windows-agents
```

**Step 5 — Remove agents from Default group (If  they were there):**
```bash
sudo /var/ossec/bin/agent_groups -r -i 001 -g default
sudo /var/ossec/bin/agent_groups -r -i 002 -g default
sudo /var/ossec/bin/agent_groups -r -i 003 -g default
```

**Step 6 — Restart agents to force sync:**

Linux agents:
```bash
sudo systemctl restart wazuh-agent
```
Windows agent:
```
Services → Wazuh → Restart
```

**Step 7 — Confirm sync in Dashboard:**
```
Agents → select agent → Configuration → verify group config is applied
```

---

## Linux Group Configuration

**File path on manager:**
```
/var/ossec/etc/shared/linux-agents/agent.conf
```

```xml
<agent_config>

  <!-- ============================================================ -->
  <!-- FILE INTEGRITY MONITORING                                     -->
  <!-- ============================================================ -->
  <syscheck>
    <disabled>no</disabled>

    <!-- Periodic full scan every 5 minutes -->
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

---

## Windows Group Configuration

**File path on manager:**
```
/var/ossec/etc/shared/windows-agents/agent.conf
```

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

> **Optional:** If Sysmon is not installed on the Windows VM, remove the Sysmon localfile block. Sysmon adds significant detection depth — worth installing in a lab context.

---

## FIM Behavior Reference

### Real-time vs Frequency scanning

| Mode | Mechanism | Use Case |
|---|---|---|
| `realtime="yes"` | inotify (Linux) / ReadDirectoryChanges (Windows) | Immediate detection on critical paths |
| `frequency` scan | Periodic snapshot + checksum compare | Catch missed/delayed changes, full-scan validation |

Both should be used together. Real-time is the primary signal; frequency scan is the safety net.

### FIM Alert Rule IDs

| Event | Rule ID | Level |
|---|---|---|
| File created | 554 | 5 |
| File modified | 550 | 7 |
| File deleted | 553 | 7 |

### Lab validation test

```bash
# On any Linux agent — ensure /etc is in syscheck config first
sudo touch /etc/wazuh_fim_test
sudo bash -c 'echo "modified" >> /etc/wazuh_fim_test'
sudo rm /etc/wazuh_fim_test
```

Then in Wazuh Dashboard:
```
Discover → search: rule.id:554 OR rule.id:550 OR rule.id:553
```

---

## Operational Notes

**Avoiding noisy alerts:**
- Do not monitor the entire filesystem (`/` or `C:\`) — it will flood the dashboard
- Add frequently-changing files to `<ignore>` (e.g. `/etc/mtab`, `/etc/resolv.conf`)
- Tune `/tmp` carefully — useful for attack detection but very noisy in practice

**Sync behavior:**
- Group config changes push on next agent check-in or restart
- Restart the agent after any group change to confirm immediately
- Dashboard → Agents → Configuration tab shows what the agent is currently running

**Duplicate directory entries:**
- A directory appearing in both `default` and a named group config causes duplicate alerts
- Once agents are moved to OS-specific groups, remove overlapping entries from `default`

---

## Mental Model

```
Agent      = sensor (collects and reports)
Group      = policy definition (what to monitor, what to collect)
Manager    = policy distributor (pushes config, processes logs)
Dashboard  = visualization layer (alerts, search, correlation)
```

---

## Next Steps (Prioritized)

| Priority | Action |
|---|---|
| 1 | Create `linux-agents` and `windows-agents` groups on the manager |
| 2 | Write and deploy `agent.conf` for each group (configs above) |
| 3 | Migrate Kali, Metasploitable2 → `linux-agents` |
| 4 | Migrate Windows VM → `windows-agents` |
| 5 | Restart all agents and confirm sync in Dashboard |
| 6 | Run FIM lab test to validate end-to-end alert flow |
| 7 | Remove default group config overlap once migration is confirmed |

---

*Part of the SOC Lab documentation series. See also: **SIEM & SOC Lab Testing Methodology** · **SIEM Lab Operational Runbook***
