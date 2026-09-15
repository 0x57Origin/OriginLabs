---
title: Home
layout: default
nav_order: 1
---

# OriginLabs

Hands-on labs I've worked through, with findings, fixes, and evidence.

## Blue Team

### Detection & Logging

1. [Sysmon Deployment](Blue_Team/Detection_Logging/Sysmon_Deployment.md) - Installed Sysmon with a config, confirmed events are landing, then read a process-create event field by field. Caught an encoded PowerShell command by filtering for `-enc` and decoded it to prove intent.
2. [Windows Audit Logging](Blue_Team/Detection_Logging/Windows_Audit_Logging.md) - Enabled Windows 11 process creation auditing, launched a process, and pulled the Event ID 4688 record that proves it was logged (Notepad).

### Hardening & Baselines

1. [SSH Hardening](Blue_Team/Hardening_Baselines/Ssh_Hardening.md) - Turned off root SSH login and added a fail2ban lockout in a Docker container, then proved it with logs and config checks.

### Response & Containment

1. [Windows Firewall Blocking](Blue_Team/Response_Containment/Windows_Firewall_Blocking.md) - Block an outbound connection with a Windows Firewall rule and confirm the traffic is stopped.

## Vulnerability Research

### Memory Safety

1. [Network Stack Overflow](Exploit_Research/Stack_Overflow_Exploitation/Network_Stack_Overflow.md) - Built a small vulnerable C server, fuzzed it with Python, and watched an oversized write overwrite the saved return address. Covers the stack frame, hex and bytes, and why the crash happens.
2. [Finding the Offset](Exploit_Research/Stack_Overflow_Exploitation/Finding_The_Offset.md) - Find the exact byte offset that lands on the saved return address, so the corruption becomes precise instead of a blind crash.

## Quick Fixes

1. [No Internet on Windows 11 VM](QuickFixes/No_Internet_VirtualBox.md) - VirtualBox VM had no route out. Traced it to the adapter sitting on a dead network plus an old static IP left over from a pfSense lab. Fixed it by putting the adapter back on the live network and flipping Windows back to DHCP.
