# Windows CIS Baseline Check (Center for Internet Security)

---

We will scan a Windows 11 machine using CIS-CAT Lite, then record the baseline score and pull the failed controls worth fixing.

CIS-CAT = Center for Internet Security Configuration Assessment Tool.

---

## Why Not Windows 11 Home

My main PC runs Windows 11 Home, but I'm using a Windows 11 Enterprise VM for this lab. Home edition matters because:

1. CIS writes the benchmark for Enterprise, so expect a low score. Some fails happen only because Home is missing features.
2. Home has no Group Policy Editor (`gpedit.msc`), so every fix has to be done manually in the registry. No fun.
3. Home is missing security features the benchmark checks for, like AppLocker and full BitLocker management, so those controls fail no matter what you do.

**Home is not meant for the CIS benchmark. Do not use it for this lab.**

---

## Lab Environment

I'm running this lab on a Windows 11 Enterprise VM. If you follow along, use a VM too and take a snapshot first, named something like `pre-CIS-baseline`.

**Why the snapshot:** some CIS settings can break applications in Windows. The snapshot is your rollback point if something stops working.

We'll also find the exact Windows 11 build, so the scores can be tied to that specific build.

---

## Download CIS-CAT Lite

Download CIS-CAT Lite from the CIS website. It asks you to fill out a registration form first, which is a little annoying, but it only takes a minute.

https://learn.cisecurity.org/cis-cat-lite

---

## Run the Assessment

Run `Assessor-GUI.exe` as an administrator. When it opens up:

1. Click the **Basic** tab.
2. Scroll down and select **CIS Microsoft Windows 11 Enterprise Benchmark v5.1.0**, then click **Add**.
3. Leave the temporary path on default.
4. Report Output Options: leave it as it is.
5. Then it will ask you if you want to start the assessment.

---

## Baseline Result

<img width="892" height="252" alt="image" src="https://github.com/user-attachments/assets/31a6562a-9003-4c14-8c4b-755560b03e3e" />

**Baseline:** 27% (99 pass, 272 fail, 2 manual) on Windows 11 Enterprise, CIS v5.1.0, Level 1. My Windows 11 is a fresh install, so the score is normal. Default Windows is easy to use, not locked down.

---

## Where the Fails Are

| Section | Pass | Fail | Score |
|---|---|---|---|
| 1 Account Policies | 2 | 8 | 20% |
| 2 Local Policies | 57 | 36 | 61% |
| 5 System Services | 17 | 2 | 89% |
| 9 Windows Defender Firewall | 0 | 23 | 0% |
| 17 Advanced Audit Policy | 10 | 17 | 37% |
| 18 Administrative Templates (Computer) | 6 | 186 | 3% |
| 19 Administrative Templates (User) | 7 | 0 | 100% |

### The 5 We Need to Fix First

1. **1.2 Account Lockout Policy.**
2. **1.1 Password Policy.**
3. **18.9.3 Include command line in process creation events.** Event ID 4688 is the Windows Security log event that fires every time a new process starts. Right now, 4688 shows that, let's say, PowerShell ran, but it does not show what it ran.
4. **18.10.26 Event Log Service, Log Size.** Default logs are small and overwrite fast, thus deleting data we need later down the line.
5. **9.3 Firewall Public Profile, logging.**
