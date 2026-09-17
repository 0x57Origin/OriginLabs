# Day 1 Task List

---

## Phase 1: Know What Exists (Week 1)

### Network Discovery
- [ ] Get a network diagram if one exists. If not, I will be making one. I already made a lab on it , if needed I can references it.
- [ ] Run a ping sweep across every subnet to find live hosts. (Run ```ip a``` on the machine which I am guessing will be kali, then look for whichever adapter is connected to the internet, wlan0 or eth0 or whatever, The IP next to it is your address and the /24 tells you the subnet to scan.)... Let's do say you see your IP as 192.168.56.103/24. That 103 is your machines address not the subnet. The SUBNET will be  192.168.56.0/24, 0 means the whole range. So the command would be like nmap -sn 192.168.56.0/24. Now we are scanning everything on the SUBNET. 
```bash
nmap -sn 192.168.56.0/24
```
- [ ] Run a full port scan against every live host and save the output.
```bash
nmap -sV -sC -p- <target_ip> -oN host_baseline.txt
```
- [ ] Document every host: IP, hostname, OS, open ports, services, and who owns it.
- [ ] Document every host: IP, hostname, OS, open ports, services, and who owns it in a text file.

### SMB & Windows Enumeration
- [ ] Run Netexec against every Windows machine to pull OS version, hostname, domain, and SMB signing status.
```bash
netexec smb <subnet>/24
```
- [ ] Check for null sessions (anonymous access).
- [ ] Pull the password policy on every Windows machine.
```bash
netexec smb <ip> -u 'user' -p 'password' --pass-pol
```
- [ ] Document every share and who has access to it.

---

## Phase 2: Know How Bad It Is (Week 1-2)

### Vulnerability Scanning
- [ ] Stand up Nessus Essentials on your machine.
- [ ] Run an **unauthenticated scan** first to see what's visible from outside.
- [ ] Run an **authenticated scan** with credentials to see everything.
- [ ] Export the report and strip the noise (informational findings, false positives).
- [ ] Rank every real finding by actual risk, not just CVSS score.
- [ ] Write your findings down: what is broken, severity, and which host.

### CIS Baseline Check
- [ ] Download CIS-CAT Lite.
- [ ] Run it against every Windows machine.
- [ ] Record the baseline score for each machine.
- [ ] Pull the failed controls worth fixing first.

### CVE Check
- [ ] Get the exact OS build number on every machine.
```powershell
[System.Environment]::OSVersion.Version
```
- [ ] Search MSRC for known CVEs on those builds.
```
https://msrc.microsoft.com/update-guide/vulnerability
```
- [ ] Check if the patch is installed.
```powershell
Get-HotFix -Id <KB_NUMBER>
```
- [ ] Document patched vs unpatched CVEs.

---

## Phase 3: Start Hardening (Week 2-3)

### Password Policy
- [ ] Set minimum password length to 14 characters.
- [ ] Enable password complexity (uppercase, numbers, symbols).
- [ ] Set account lockout threshold to 5 attempts.
- [ ] Set lockout duration to 15 minutes.

### Network
- [ ] Disable SMBv1 on every Windows machine.
- [ ] Enable SMB signing on every Windows machine.
- [ ] Disable LLMNR and NetBIOS where possible (stops Responder attacks).
- [ ] Block unnecessary outbound connections with firewall rules.
- [ ] Enable firewall logging on all profiles (Domain, Private, Public).

### RDP & Remote Access
- [ ] Disable RDP on machines that don't need it.
- [ ] Enable Network Level Authentication on machines that do need it.
- [ ] Enable MFA on RDP if the environment supports it.

### Windows Hardening
- [ ] Enable process creation auditing (Event ID 4688) on every machine.
- [ ] Enable command line logging in 4688 events.
- [ ] Deploy Sysmon with a config on every Windows machine.
- [ ] Increase Security event log size to at least 196,608 KB.

---

## Phase 4: Build Visibility (Week 3-4)

### Logging
- [ ] Confirm Sysmon is running and events are landing on every host.
- [ ] Confirm Windows Security logs are capturing 4688 (process creation) and 4625 (failed logon).
- [ ] Set up a central log collector (even a simple one to start).
- [ ] Ship Windows logs to the collector.
- [ ] Ship Linux logs (auditd or journald) to the collector.

### Detection
- [ ] Write a detection rule for encoded PowerShell (-enc).
- [ ] Write a detection rule for failed logon spikes (brute force).
- [ ] Write a detection rule for new local admin accounts being created (Event ID 4720, 4732).
- [ ] Write a detection rule for cleared event logs (Event ID 1102).

---

## Phase 5: Document Everything (Ongoing)

### What to Write Down
- [ ] Asset inventory (every device on the network).
- [ ] Vulnerability findings (what's broken, severity, status).
- [ ] What you hardened and when.
- [ ] What you couldn't fix and why (this is your POA&M).
- [ ] Incident log (anything suspicious you investigated).

### Reports to Have Ready
- [ ] **Baseline Report** - what the environment looked like on day 1.
- [ ] **Findings Report** - vulnerabilities found, ranked by risk.
- [ ] **Remediation Report** - what was fixed, before and after proof.
- [ ] **POA&M** - what's still open, who owns it, and target fix date.

---

## Quick Reference Commands

```bash
# Ping sweep
nmap -sn 192.168.1.0/24

# Full port scan with service detection
nmap -sV -sC -p- <ip> -oN output.txt

# SMB fingerprint
netexec smb <ip>

# SMB shares
netexec smb <ip> -u 'user' -p 'pass' --shares

# Password policy
netexec smb <ip> -u 'user' -p 'pass' --pass-pol

# Check patch installed
Get-HotFix -Id KB5065426

# Windows build number
[System.Environment]::OSVersion.Version

# Start Nessus
/bin/systemctl start nessusd.service
```

---

## Priority Order (If You Can Only Do One Thing at a Time)

1. Asset inventory - you can't protect what you don't know exists.
2. Vulnerability scan - find the worst problems first.
3. Password policy - easiest win, biggest impact.
4. Patch the critical CVEs - close the known doors.
5. Turn on logging - you're blind without it.
6. Write it all down - if it's not documented it didn't happen.
