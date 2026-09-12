# Windows Logging & Event Detection
---
## We will enable windows audit policy on a Windows 11 VM and also detect failed logins attempts - NIST 800-171: 3.3.1
---
### We need to check if the audit logging is even on first! (audit policy)
-> Open PowerShell @admin and type: auditpol /get /category:*
    /get /category:* -> This just dumps all the audit policy to the terminal

```
PS C:\WINDOWS\system32> auditpol /get /category:*
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
PS C:\WINDOWS\system32>
```
---
What jumps off right away is the Detailed Audit -> Process Creation: No Auditing. So right now windows is not logging when a new process meaning program launches. Something attackers loveeee. Any malware can run on this windows and nobody would know. Turn this on asap.

Now we will turn on the Process Creation on and then launch a program and see if it logs or not.

Okay in admin shell we have to type this: auditpol /set /subcategory:"Process Creation" /success:enable

<img width="1915" height="137" alt="image" src="https://github.com/user-attachments/assets/5dbafc68-1a22-4864-8594-7769bb46b108" />

After that we will dump the audit policy again and see if it's there now using -> auditpol /get /category:*

<img width="1917" height="242" alt="image" src="https://github.com/user-attachments/assets/1afe6f50-3aa8-4d87-9f1e-5d39a4130e03" />

Or we could do straight process creation dump -> auditpol /get /subcategory:"Process Creation"
```
PS C:\WINDOWS\system32> auditpol /get /subcategory:"Process Creation"
System audit policy
Category/Subcategory                      Setting
Detailed Tracking
  Process Creation                        Success
PS C:\WINDOWS\system32>
```
Success!!!
