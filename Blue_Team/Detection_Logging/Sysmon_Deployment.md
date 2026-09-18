# Sysmon Deployment

**Deep Windows process and network visibility for threat data gathering (NIST 800-171: 3.3.1)**

---

## 1. Download Sysmon

My Windows VM already gathers baseline process creation, so we do not need to mess with audit policy. We can go straight to downloading Sysmon (System Monitor) from Windows Sysinternals, right out of PowerShell.

```powershell
Invoke-WebRequest -Uri "https://download.sysinternals.com/files/Sysmon.zip" -OutFile "Sysmon.zip"
Expand-Archive Sysmon.zip -DestinationPath Sysmon
```

**Breaking it down:**

| Piece | What it does |
|---|---|
| `Uri` | Uniform Resource Identifier, the address we are pulling from |
| `Expand-Archive` | Unzips a zip file |
| `Sysmon.zip` | The source zip file |
| `-DestinationPath` | The folder to dump all the Sysmon files into. If the folder does not exist, it gets created |

> **Tip:** If the username on the machine has a space in it, like `Random User`, `cd` can throw an error in PowerShell. The fix is to wrap the path in quotes: `cd "C:\Users\Random User\Desktop"`. Just use `""` and it fixes itself.

---

## 2. Install the Config

Sysmon is a CLI program. Open PowerShell, `cd` into the Sysmon folder, and run `.\Sysmon64.exe`.

Bare Sysmon logs nothing useful on its own. We install a config file so it watches for what we actually want.

Download the config XML into the same folder:

```powershell
Invoke-WebRequest -Uri "https://raw.githubusercontent.com/SwiftOnSecurity/sysmon-config/master/sysmonconfig-export.xml" -OutFile "sysmonconfig.xml"
```

Now load it. The `-accepteula` flag accepts the EULA (End User License Agreement, the legal terms you accept to use the software) so it does not stop and prompt you:

```powershell
.\Sysmon64.exe -accepteula -i sysmonconfig.xml
```

**Output:**

```text
Loading configuration file with schema version 4.50
Sysmon schema version: 4.91
Configuration file validated.
Sysmon64 installed.
SysmonDrv installed.
Starting SysmonDrv.
SysmonDrv started.
Starting Sysmon64..
Sysmon64 started.
```

Sysmon now logs every process that starts and every connection made, writing each one as an event we can search for.

---

## 3. Confirm Events Are Landing

```powershell
Get-WinEvent -LogName "Microsoft-Windows-Sysmon/Operational" -MaxEvents 5
```

**Output:**

```text
   ProviderName: Microsoft-Windows-Sysmon

TimeCreated                      Id LevelDisplayName Message
-----------                      -- ---------------- -------
9/14/2026 1:31:15 AM              1 Information      Process Create:...
9/14/2026 1:31:15 AM              1 Information      Process Create:...
9/14/2026 1:31:15 AM              4 Information      Sysmon service state changed:...
9/14/2026 1:31:14 AM             16 Information      Sysmon config state changed:...
```

---

## 4. Read a Process Create Event

Let's open an Event ID 1 and read it to see what process was created.

```powershell
Get-WinEvent -LogName "Microsoft-Windows-Sysmon/Operational" -MaxEvents 1 -FilterXPath "*[System[EventID=1]]" | Format-List
```

`-FilterXPath` grabs only EventID 1. `Format-List` prints the full message instead of a cramped table.

We get a lot of information on the screen, but let's start with the `Image` value:

```text
Image: C:\Windows\System32\wbem\unsecapp.exe
```

That is a normal Microsoft Windows WMI binary. WMI (Windows Management Instrumentation) is built-in infrastructure in Windows that lets administrators and developers manage system data, configurations, and operations both locally and remotely. For our investigation, `Image` is showing us the path to it.

There are two more key/value pairs worth looking at.

### CommandLine: how it was launched

```text
CommandLine: C:\WINDOWS\system32\wbem\unsecapp.exe -Embedding
```

Args (arguments) are the extra text passed to a program when it runs, and args matter for spotting attacks. This is where the suspicious stuff shows up. It is the field where you can see if a process is encoded, obscured, or downloading something.

### ParentImage: what launched it

```text
ParentImage: C:\Windows\System32\svchost.exe
```

This is the heart of the parent/child relationship, and attacks show up as weird chains.

Example: `winword.exe -> powershell.exe`. Word should not spawn a shell, so that is a malicious macro. A macro is a small script embedded inside a Microsoft Office file, and attackers abuse this all the time.

Here, `svchost.exe -> unsecapp.exe` is normal Windows behavior.

---

## 5. Purple Scenario: Red Generates It, Blue Hunts It

**Scenario:** create an encoded PowerShell command, then find it in the Sysmon log by its `CommandLine`.

```powershell
powershell -enc VwByAGkAdABlAC0ASABvAHMAdAAgAGgAaQA=
```

That is Base64. Base64 turns files or images into text so they can be sent easily over the internet, and it can write any data using 64 text characters. So a whole command becomes a safe-looking string. `-enc` is just there to hide what PowerShell is running.

> **Note:** This is a safe detection test. The payload decodes to a harmless `Write-Host hi`.

### Find it in Sysmon

```powershell
Get-WinEvent -LogName "Microsoft-Windows-Sysmon/Operational" -MaxEvents 5
```

**Output:**

```text
   ProviderName: Microsoft-Windows-Sysmon

TimeCreated                      Id LevelDisplayName Message
-----------                      -- ---------------- -------
9/14/2026 1:51:49 AM             11 Information      File created:...
9/14/2026 1:51:49 AM              1 Information      Process Create:...
9/14/2026 1:51:47 AM              1 Information      Process Create:...
9/14/2026 1:46:21 AM              3 Information      Network connection detected:...
9/14/2026 1:35:35 AM             22 Information      Dns query:...
```

Checking ID 1, I see:

```text
CommandLine: "C:\WINDOWS\System32\WindowsPowerShell\v1.0\powershell.exe" -enc VwByAGkAdABlAC0ASABvAHMAdAAgAGgAaQA=
```

### Decode it

```powershell
[System.Text.Encoding]::Unicode.GetString([Convert]::FromBase64String("VwByAGkAdABlAC0ASABvAHMAdAAgAGgAaQA="))
```

Unicode is a giant list that gives every letter, number, and symbol its own code, so computers everywhere show text the same way.

**Output:**

```text
Write-Host hi
```

---

## 6. Filter It Better

Instead of going through events one by one, filter straight for the encoded ones:

```powershell
Get-WinEvent -LogName "Microsoft-Windows-Sysmon/Operational" -FilterXPath "*[System[EventID=1]]" -MaxEvents 10 | Where-Object { $_.Message -like "*-enc*" } | Format-List
```

**Breaking it down:**

| Piece | What it does |
|---|---|
| `Where-Object` | A filter. Keeps only items that pass a test |
| `{ }` | The test goes inside these braces |
| `$_` | "The current item" being checked (one event) |
| `.Message` | The field on that event you want to look at |
| `-like` | Compare with wildcard matching |
| `"*-enc*"` | The pattern. `*` means "anything," so this matches any message containing `-enc` anywhere |

Basically, `Where-Object` is running a test to see if the item has that key/value pair.

---

## Conclusion

We covered how to download the Sysmon CLI from Sysinternals and unzip it into a folder, then install it with a config so it actually logs something useful. We confirmed events were landing, then read a process create event field by field (`Image`, `CommandLine`, `ParentImage`). Finally, we generated a fake encoded PowerShell string, Sysmon caught it, and we decoded it to prove intent.

This scenario satisfies **NIST 800-171 3.3.1** by creating and retaining detailed process and network audit records at the system level.
