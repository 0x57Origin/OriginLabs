# OriginLabs

Hands-on labs I've worked through, with findings, fixes, and evidence.

## Labs - Blue Team
- [SSH Hardening](Labs/Ssh_Hardening.md) - Turn off root SSH login and add fail2ban lockout in a Docker container, then I tried to prove it works via logs and configuration checks.

- [Windows Audit Logging](Labs/Windows_Audit_Logging.md) - Enable Windows 11 process creation auditing, launch a process, and get the Event ID 4688 record that proves it was logged. It was done with Notepad as proof.

- [Sysmon Deployment](Labs/Sysmon_Deployment.md) - I installed Sysmon with a config, confirm events are landing, then read a process-create event field by field. Ran an encoded PowerShell command, caught it in the log by filtering for `-enc`, and decoded it to prove intent. It is a beginner version but it does cover a lot of the Sysmon commands. P.S it logs more than you would think. 

## Labs - Red / Exploitation Research

- [Network Stack Overflow](Exploit_Research/Stack_Overflow_Exploit/Network_Stack_Overflow.md) - Built a vulnerable C server then fuzzing it with Python to see how an oversized write smashes the saved return address. It does cover a lot of information that is required for exploit/fuzzing research like buffer overflows, the stack frame, hex/bytes, and why crashes happen.

## Quick Fixes

- [No Internet on Windows 11 VM](QuickFixes/No_Internet_VirtualBox.md) - VirtualBox VM had no route out. I traced it to the adapter settings sitting on a dead network plus an old static IP address left over from a pfSense lab. Fixed it by setting adapter back on live network and flipping Windows back to DHCP. 
