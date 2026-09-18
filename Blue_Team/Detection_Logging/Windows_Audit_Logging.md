# Windows Logging and Event Detection

**Lab:** Enabling Windows 11 audit policy and detecting process creation events (Event ID 4688)
**Control mapping:** NIST SP 800-171, 3.3.1 (Create and retain system audit logs and records)

---

## Objective

Verify whether Windows is logging process creation, enable the audit subcategory if it is disabled, and confirm that a newly launched process generates a Security log entry (Event ID 4688).

**Environment:** Windows 11, local workstation, PowerShell running as Administrator.

---

## Step 1: Check the current audit policy

Before changing anything, confirm what is being audited. Open PowerShell as Administrator and run:

```powershell
auditpol /get /category:*
```

`/get /category:*` dumps every audit category and subcategory to the terminal.

**Output:**

```
System audit policy
Category/Subcategory                      Setting
System
  Security System Extension               No Auditing
  System Integrity                        Success and Failure
  IPsec Driver                            No Auditing
  Other System Events                     Success and Failure
  Security State Change                   Success
Logon/Logoff
  Logon                                   Success and Failure
  Logoff                                  Success
  Account Lockout                         Success
  IPsec Main Mode                         No Auditing
  IPsec Quick Mode                        No Auditing
  IPsec Extended Mode                     No Auditing
  Special Logon                           Success
  Other Logon/Logoff Events               No Auditing
  Network Policy Server                   Success and Failure
  User / Device Claims                    No Auditing
  Group Membership                        No Auditing
  Access Rights                           No Auditing
Object Access
  File System                             No Auditing
  Registry                                No Auditing
  Kernel Object                           No Auditing
  SAM                                     No Auditing
  Certification Services                  No Auditing
  Application Generated                   No Auditing
  Handle Manipulation                     No Auditing
  File Share                              No Auditing
  Filtering Platform Packet Drop          No Auditing
  Filtering Platform Connection           No Auditing
  Other Object Access Events              No Auditing
  Detailed File Share                     No Auditing
  Removable Storage                       No Auditing
  Central Policy Staging                  No Auditing
Privilege Use
  Non Sensitive Privilege Use             No Auditing
  Other Privilege Use Events              No Auditing
  Sensitive Privilege Use                 No Auditing
Detailed Tracking
  Process Creation                        No Auditing
  Process Termination                     No Auditing
  DPAPI Activity                          No Auditing
  RPC Events                              No Auditing
  Plug and Play Events                    No Auditing
  Token Right Adjusted Events             No Auditing
Policy Change
  Audit Policy Change                     Success
  Authentication Policy Change            Success
  Authorization Policy Change             No Auditing
  MPSSVC Rule-Level Policy Change         No Auditing
  Filtering Platform Policy Change        No Auditing
  Other Policy Change Events              No Auditing
Account Management
  Computer Account Management             No Auditing
  Security Group Management               Success
  Distribution Group Management           No Auditing
  Application Group Management            No Auditing
  Other Account Management Events         No Auditing
  User Account Management                 Success
DS Access
  Directory Service Access                No Auditing
  Directory Service Changes               No Auditing
  Directory Service Replication           No Auditing
  Detailed Directory Service Replication  No Auditing
Account Logon
  Kerberos Service Ticket Operations      No Auditing
  Other Account Logon Events              No Auditing
  Kerberos Authentication Service         No Auditing
  Credential Validation                   No Auditing
```

### Finding

Under **Detailed Tracking**, `Process Creation` is set to **No Auditing**.

That means Windows is not recording when a new program launches. Without it, a process can execute on this host and leave no record in the Security log, which removes one of the most useful sources of evidence for detection and incident response. This should be enabled.

---

## Step 2: Enable process creation auditing

In the Administrator shell:

```powershell
auditpol /set /subcategory:"Process Creation" /success:enable
```

<img width="1915" height="137" alt="Enabling the Process Creation audit subcategory" src="https://github.com/user-attachments/assets/5dbafc68-1a22-4864-8594-7769bb46b108" />

---

## Step 3: Verify the change

Dump the full policy again:

```powershell
auditpol /get /category:*
```

<img width="1917" height="242" alt="Audit policy after enabling Process Creation" src="https://github.com/user-attachments/assets/1afe6f50-3aa8-4d87-9f1e-5d39a4130e03" />

Or query just the one subcategory:

```powershell
auditpol /get /subcategory:"Process Creation"
```

**Output:**

```
System audit policy
Category/Subcategory                      Setting
Detailed Tracking
  Process Creation                        Success
```

The setting is now **Success**.

---

## Step 4: Generate and locate an event

Launch Notepad to create a process, then pull the five most recent 4688 events from the Security log:

```powershell
Get-WinEvent -FilterHashtable @{LogName='Security'; ID=4688} -MaxEvents 5 |
    Format-List TimeCreated, Message
```

**Output (most recent event):**

```
TimeCreated : 9/11/2026 8:39:13 PM
Message     : A new process has been created.

              Creator Subject:
                Security ID:            S-1-5-21-3841895432-4071795221-4240210549-1001
                Account Name:           Someone Unknown
                Account Domain:         DESKTOP-APE8POJ
                Logon ID:               0x480C1

              Target Subject:
                Security ID:            S-1-0-0
                Account Name:           -
                Account Domain:         -
                Logon ID:               0x0

              Process Information:
                New Process ID:         0x18cc
                New Process Name:       C:\Program Files\WindowsApps\Microsoft.WindowsNotepad_11.2607.14.0_x64__8wekyb3d8bbwe\Notepad\Notepad.exe
                Token Elevation Type:   TokenElevationTypeFull (2)
                Mandatory Label:        S-1-16-12288
                Creator Process ID:     0x11ec
                Creator Process Name:   C:\Program Files\WindowsApps\Microsoft.WindowsNotepad_11.2607.14.0_x64__8wekyb3d8bbwe\Notepad\Notepad.exe
                Process Command Line:
```

The event was captured. The launch of Notepad is now recorded with the account that started it, the full image path, the new process ID, and the parent (creator) process ID.

---

## Notes

**Token Elevation Type** describes the token assigned to the new process under User Account Control:

| Type | Meaning |
| --- | --- |
| Type 1 | Full token, no privileges removed or groups disabled. Used only when UAC is disabled, or for the built in Administrator or a service account. |
| Type 2 | Elevated token. Used when UAC is enabled and the program was started with "Run as administrator", or the application always requires elevation and the user is in the Administrators group. |
| Type 3 | Limited token, administrative privileges removed and administrative groups disabled. Used when UAC is enabled and the application does not require elevation. |

**Process Command Line is empty.** Enabling the audit subcategory alone does not record arguments. Command line capture is a separate setting (`Include command line in process creation events`, under Computer Configuration > Administrative Templates > System > Audit Process Creation). Without it, an event shows that `powershell.exe` ran but not what it ran, which is the detail that usually matters in detection work.

**Creator process.** In this capture the creator process is also Notepad, which is expected for modern packaged apps that relaunch themselves rather than staying as a child of the shell.

---

## Revert

To return the host to its original state:

```powershell
auditpol /set /subcategory:"Process Creation" /success:disable
```

---

## Takeaways

- Audit policy defaults are not sufficient for detection. Process creation was off out of the box.
- Event ID 4688 gives account, image path, process ID, and parent process ID, which is the base data for process lineage analysis.
- Enabling the subcategory is only half the job. Turn on command line logging as well, or the events will be far less useful.
- This maps to NIST SP 800-171 3.3.1, which requires audit records to be created and retained to support monitoring, analysis, investigation, and reporting.
