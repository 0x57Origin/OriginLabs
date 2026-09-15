# OriginLabs

Hands-on labs I've worked through, with findings, fixes, and evidence.

## Blue Team

### Detection & Logging

- [Windows Audit Logging](Blue_Team/Detection_Logging/Windows_Audit_Logging.md) - Enable Windows 11 process creation auditing, launch a process, and get the Event ID 4688 record that proves it was logged. Done with Notepad as proof.

- [Sysmon Deployment](Blue_Team/Detection_Logging/Sysmon_Deployment.md) - Installed Sysmon with a config, confirmed events are landing, then read a process-create event field by field. Ran an encoded PowerShell command, caught it by filtering for `-enc`, and decoded it to prove intent. Beginner version, but it covers a lot of the Sysmon commands. P.S. it logs more than you would think.

### Hardening & Baselines

- [SSH Hardening](Blue_Team/Hardening_Baselines/Ssh_Hardening.md) - Turn off root SSH login and add a fail2ban lockout in a Docker container, then prove it works with logs and config checks.

### Response & Containment

- [Windows Firewall Blocking](Blue_Team/Response_Containment/Windows_Firewall_Blocking.md) - Block an outbound connection with a Windows Firewall rule and confirm the traffic is actually stopped.

## Vulnerability Research

### Memory Safety

- [Network Stack Overflow](Exploit_Research/Stack_Overflow_Exploitation/Network_Stack_Overflow.md) - Built a small vulnerable C server, then fuzzed it with Python to see how an oversized write overwrites the saved return address. Covers a lot of the groundwork for this kind of research: buffer overflows, the stack frame, hex and bytes, and why crashes happen.

- [Finding the Offset](Exploit_Research/Stack_Overflow_Exploitation/Finding_The_Offset.md) - Find the exact byte offset that lands on the saved return address, so the corruption becomes precise instead of a blind crash.

## Quick Fixes

- [No Internet on Windows 11 VM](QuickFixes/No_Internet_VirtualBox.md) - VirtualBox VM had no route out. Traced it to the adapter sitting on a dead network plus an old static IP left over from a pfSense lab. Fixed it by putting the adapter back on the live network and flipping Windows back to DHCP.