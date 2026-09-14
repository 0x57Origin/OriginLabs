# OriginLabs

Hands-on labs I've worked through, with findings, fixes, and evidence.

## Labs - Blue Team
- [SSH Hardening](Labs/Ssh_Hardening.md) - Turn off root SSH login and add fail2ban lockout in a Docker container, then I tried to prove it works via logs and configuration checks.

- [Windows Audit Logging](Labs/Windows_Audit_Logging.md) - Enable Windows 11 process creation auditing, launch a process, and get the Event ID 4688 record that proves it was logged. It was done with Notepad as proof.

## Labs - Red / Exploitation Research

- [Network Stack Overflow](Exploit_Research/Network_Stack_Overflow.md) - Building a vulnerable C server then fuzzing it with Python to see how an oversized write smashes the saved return address. It does cover a lot of information that is required for exploit/fuzzing research like buffer overflows, the stack frame, hex/bytes, and why crashes happen.
