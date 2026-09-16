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

Mine is -> **Host:** Windows 11 Enterprise Evaluation, Version 25H2 (OS Build 26200.6584)

---

## Download CIS-CAT Lite

Download CIS-CAT Lite from the CIS website. It asks you to fill out a registration form first, which is a little annoying, but it only takes a minute.

https://learn.cisecurity.org/cis-cat-lite

---

## Run the Assessment

Run `Assessor-GUI.exe` as an administrator. When it opens up:

1. Click the **Basic** tab.
2. Scroll down and select **CIS Microsoft Windows 11 Enterprise Benchmark v5.1.0**, pick **Level 1 (L1)**, then click **Add**.
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

### The 5 Settings We Should Fix First

1. **1.2 Account Lockout Policy.**
2. **1.1 Password Policy.**
3. **18.9.3 Include command line in process creation events.** Event ID 4688 is the Windows Security log event that fires every time a new process starts. Right now, 4688 shows that, let's say, PowerShell ran, but it does not show what it ran.
4. **18.10.26 Event Log Service, Log Size.** Default logs are small and overwrite fast, thus deleting data we need later down the line.
5. **9.3 Firewall Public Profile, logging.**

### Profiles

1. Profile Level 1 is normal scans for most corporate computers.
2. Level 2 is much stricter for high-security systems and can break things.
3. The BitLocker (BL) version just adds disk encryption on top of everything.

---

## Assessment Results

#### Account Lockout Duration

Now turn on **Failures Only**, then scroll to 1.2.1 and click to expand, then click -> **Show Assessment Evidence**.

<img width="882" height="332" alt="image" src="https://github.com/user-attachments/assets/40278795-6d6c-49cf-8900-3696761132fd" />

See, the actual value is 600s = 10 minutes. CIS wants 15 or more.

#### Password Policy

Let's click 1.1.4 Minimum password length. Now same thing, click on **Show Assessment Evidence**. CIS wants 14 or more for the password length, but the VM is set to 0. The evidence also shows the password complexity rule is off and password history is 0.

<img width="867" height="277" alt="image" src="https://github.com/user-attachments/assets/b61d3bb0-fb84-429e-9daf-17d44057fb49" />

#### Include Command Line in Process Creation Events

Let's click on 18.9.3.1 and then click on **Show Assessment Evidence**. There is a switch to turn it off and on for this, but in our Windows 11 VM that switch does not exist. CIS wants it turned on.

<img width="885" height="432" alt="image" src="https://github.com/user-attachments/assets/fad280f2-824c-4117-8433-8abecb99b7e3" />

See, it says no matching system items were found.

#### Event Log Service

Click on 18.10.26 Event Log Service, and if that does not work, click 18.10.26.2 Security. The title will say -> 18.10.26.2.2 Ensure 'Security: Specify the maximum log file size (KB)' is set to 'Enabled: 196,608 or greater' -> about 192 MB or more. The normal Windows default size is about 20 MB. To prove it, let's use PowerShell.

```powershell
wevtutil gl Security
```

- `wevtutil` = Windows Event Utility
- `gl` = get-log -> shows log settings, like its max size.

**Result:**

```
PS C:\WINDOWS\system32> wevtutil gl Security
name: Security
enabled: true
type: Admin
owningPublisher:
isolation: Custom
channelAccess: O:BAG:SYD:(A;;0xf0005;;;SY)(A;;0x5;;;BA)(A;;0x1;;;S-1-5-32-573)
logging:
  logFileName: %SystemRoot%\System32\Winevt\Logs\Security.evtx
  retention: false
  autoBackup: false
  maxSize: 20971520
publishing:
  fileMax: 1
PS C:\WINDOWS\system32>
```

See `maxSize: 20971520` -> 20,971,520 bytes = 20 MB.

<img width="950" height="260" alt="image" src="https://github.com/user-attachments/assets/b259214c-e656-485c-8a8d-a074d4c28dc2" />

#### Firewall Public Profile

Last one, which is 9.3 Firewall Public Profile. Title: 9.3.8 Ensure 'Windows Firewall: Public: Logging: Log dropped packets' is set to 'Yes'.

<img width="936" height="285" alt="image" src="https://github.com/user-attachments/assets/b1ba81d1-2d2e-4bab-9c1d-a6fa00496ff4" />

The setting is not configured, so Windows is not logging dropped packets on the Public profile. CIS wants it on.

**Public Profile** - It is the firewall mode on untrusted networks, like a small coffee shop Wi-Fi or the airport. CIS wants it logging dropped packets so you have a record of who tried to connect and got blocked.

CIS is just a public nonprofit that publishes free security checks, built by agreement among experts from the gov, companies & schools. Nobody is legally bound by it, but most auditors use it. In DoD they usually use DISA STIG, which is the government version of the same idea.

---

## Findings

| Rule | Title | Expected | Actual |
|---|---|---|---|
| 1.1.4 | Minimum password length | 14 or more characters | 0 characters |
| 1.2.1 | Account lockout duration | 15 or more minutes | 10 minutes |
| 1.2.2 | Account lockout threshold | 5 or fewer invalid attempts (not 0) | 10 attempts |
| 1.2.4 | Reset account lockout counter after | 15 or more minutes | 10 minutes |
| 18.9.3.1 | Include command line in process creation events | Enabled | Not configured |
| 18.10.26.2.2 | Security log maximum size | 196,608 KB (about 192 MB) or more | 20 MB (20,971,520 bytes, Windows default) |
| 9.3.8 | Public profile: log dropped packets | Yes | Not configured |











