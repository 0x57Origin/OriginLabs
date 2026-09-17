# OriginLabs

Hands-on labs I've worked through, with findings, fixes, and evidence.

## Blue Team

### Detection & Logging

1\. [Sysmon Deployment](Blue_Team/Detection_Logging/Sysmon_Deployment.md) - Installed Sysmon with a config, confirmed events are landing, then read a process-create event field by field. Caught an encoded PowerShell command by filtering for `-enc` and decoded it to prove intent.

2\. [Windows Audit Logging](Blue_Team/Detection_Logging/Windows_Audit_Logging.md) - Enabled Windows 11 process creation auditing, launched a process, and pulled the Event ID 4688 record that proves it was logged (Notepad).

### Hardening & Baselines

1\. [SSH Hardening](Blue_Team/Hardening_Baselines/Ssh_Hardening.md) - Turned off root SSH login and added a fail2ban lockout in a Docker container, then proved it with logs and config checks.

2\. [CIS Baseline Check on Windows](Blue_Team/Hardening_Baselines/Windows_CIS_Baseline_Check.md) - Scanned a Windows 11 Enterprise VM with CIS-CAT Lite, received a 27% baseline score, and pulled the evidence for 7 failed controls.

### Response & Containment

1\. [Windows Firewall Blocking](Blue_Team/Response_Containment/Windows_Firewall_Blocking.md) - Blocked an outbound connection with a Windows Firewall rule and confirmed the traffic is stopped. Source: Firewall Logging & Sysmon Event ID 3.

## Vulnerability Research

### Stack Overflow Exploitation Research

1\. [Network Stack Overflow](Exploit_Research/Stack_Overflow_Exploitation/Network_Stack_Overflow.md) - Built a small vulnerable C server, fuzzed it with Python, and watched an oversized write overwrite the saved return address. Covers the stack frame, hex and bytes, and why the crash happens.

2\. [Finding the Offset](Exploit_Research/Stack_Overflow_Exploitation/Finding_The_Offset.md) - Found the exact byte offset that lands on the saved return address using a Metasploit cyclic pattern and gdb. Offset confirmed at 72 bytes.

## Red Team / Pentest

### Lab Setup

1\. [Building the Isolated Lab Network](Pentesting/Lab_Setup/Isolated_Lab.md) - Built a host-only network in VirtualBox, deployed Juice Shop in Docker, confirmed Kali can reach the Windows 11 VM on 192.168.56.0/24, and set up HTB for AD labs.

### Reconnaissance & Asset Discovery

1\. [Internal Asset Discovery](Pentesting/Reconnaissance_&_Asset_Discovery/Internal_asset_discovery.md) - Full Nmap port scan against a Windows 11 VM, confirmed 3 open ports (135, 139, 445), pulled the hostname and OS build with Netexec, and documented the asset inventory baseline.

2\. [Service Fingerprinting](Pentesting/Reconnaissance_&_Asset_Discovery/Service_Fingerprinting.md) - Enabled RDP and WinRM on the Windows 11 VM, then I ran Nmap NSE scripts against both services, enumerated SMB shares and password policy with Netexec, and documented all the findings including a patched CVE (CVE-2025-21293). CVE also stands for Common Vulnerabilities and Exposures. 


## Quick Fixes

1\. [No Internet on Windows 11 VM](QuickFixes/No_Internet_VirtualBox.md) - VirtualBox VM had no route out. Traced it to the adapter sitting on a dead network plus an old static IP left over from a pfSense lab. Fixed it by putting the adapter back on the live network and flipping Windows back to DHCP.

## Quick Labs

1\. [Is Windows SSH Exposed](QuickLabs/Windows_SSH_Exposed.md) - Simple checks to gather information about the Windows SSH: Exposed or Not Exposed.

## Cheat Sheet

1\. [Firewall Commands](Cheat_Sheet/FireWall.md) - Basic Firewall Commands.

2\. [Sysmon Commands](Cheat_Sheet/Sysmon_Commands.md) - Basic Sysmon Commands.
