# Sysmon Deployment - Looking at the Deep Windows Process & Network Visibility for Threat Data Gathering (NIST 800-171: 3.3.1)

---

My Windows VM already gathers baseline process creation, so we do not need to mess with audit policy. So we can just go ahead and download Sysmon (System Monitor). We can download it straight from PowerShell into a folder from Windows Sysinternals.

PowerShell:

```powershell
Invoke-WebRequest -Uri "https://download.sysinternals.com/files/Sysmon.zip" -OutFile "Sysmon.zip"
Expand-Archive Sysmon.zip -DestinationPath Sysmon
```

-Uri - Uniform Resource Locator. Also, if you ever want to change directory and the username has a space in it, like -> Random User -> you can use -> cd "Location" and it will not cause any error issues in PowerShell.

Expand-Archive - Unzips a zip file.
Sysmon.zip - The source zip file.
-DestinationPath - The folder to dump all those Sysmon.zip items or files, and if the Sysmon folder doesn't exist it will create one.

---

Also, Sysmon is a CLI program. Open PowerShell and CD into the Sysmon folder and run it there -> .\Sysmon64.exe. Bare Sysmon itself logs nothing useful at all. But we can install a config file to watch for what we want.

## Install Config

We can just install it in the same folder using the eula file. Now, what is eula? End User License Agreement, the legal terms you accept to use the software. Let's download that xml file now.

```powershell
Invoke-WebRequest -Uri "https://raw.githubusercontent.com/SwiftOnSecurity/sysmon-config/master/sysmonconfig-export.xml" -OutFile "SysmonConfig.xml"
```

```
.\Sysmon64.exe -accepteula -i sysmonconfig.xml


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

Sysmon now logs every process that starts and every connection made, writing each as an event we can search for. Now confirm events are landing. Open Event Viewer in the Sysmon log:

```
Get-WinEvent -LogName "Microsoft-Windows-Sysmon/Operational" -MaxEvents 5

PS C:\Users\Someone Unknown\Desktop\Sysmon\Sysmon> Get-WinEvent -LogName "Microsoft-Windows-Sysmon/Operational" -MaxEvents 5


   ProviderName: Microsoft-Windows-Sysmon

TimeCreated                      Id LevelDisplayName Message
-----------                      -- ---------------- -------
9/14/2026 1:31:15 AM              1 Information      Process Create:...
9/14/2026 1:31:15 AM              1 Information      Process Create:...
9/14/2026 1:31:15 AM              4 Information      Sysmon service state changed:...
9/14/2026 1:31:14 AM             16 Information      Sysmon config state changed:...


PS C:\Users\Someone Unknown\Desktop\Sysmon\Sysmon>
```

Well, now let's open an ID and read the log. Let's go with ID 1 to see what process was created.

```
Get-WinEvent -LogName "Microsoft-Windows-Sysmon/Operational" -MaxEvents 1 -FilterXPath "*[System[EventID=1]]" | Format-List
```

-FilterXPath grabs only EventID 1; Format-List prints the full message instead of a cramped table.

Well, we got a lot of information on the screen, but let's look at the Image value. Image: C:\Windows\System32\wbem\unsecapp.exe.

It is a normal Microsoft Windows WMI -> Windows Management Instrumentation. What is it? It is a built-in infrastructure in Windows operating systems that allows administrators and developers to manage system data, configurations, and operations both locally and remotely. Well, for our investigation the Image is showing us the path to it. There are 2 more key:value pairs to look at.

1. CommandLine - How it was launched. (args matter for spotting attacks) ... args - Arguments. The extra text passed to a program when it runs. CommandLine: C:\WINDOWS\system32\wbem\unsecapp.exe -Embedding -> This is where the suspicious stuff would appear. This is the field where you can see if any process is encoded or obscured, or even downloading something.
2. ParentImage - What launched it. ParentImage: C:\Windows\System32\svchost.exe -> Now we can see what launched it. Heart of the parent/child relationship; attacks show up as weird chains. Example -> winword.exe -> powershell.exe. Word should not spawn a shell, that is a malicious macro. Here, svchost -> unsecapp is normal Windows behavior. A macro is a small script embedded inside of a Microsoft Office file. Attackers abuse this all the time.

---

# Purple Scenario: Red Generates It, Blue Hunts It

Scenario: Let's create an encoded PowerShell command, then find it in the Sysmon log by its CommandLine.

```
powershell -enc VwByAGkAdABlAC0ASABvAHMAdAAgAGgAaQA=
```

That is base64. Base64 turns files or images into text so they can be easily sent over the internet. Now, base64 can be used to write any data with 64 text characters. So a whole command becomes a safe-looking string. -enc is just to hide what PowerShell is running.

> Note: This is a safe detection test. The payload decodes to a harmless `Write-Host hi`.

---

Let's find it in our Sysmon.

```
PS C:\Users\Someone Unknown\Desktop\Sysmon\Sysmon> Get-WinEvent -LogName "Microsoft-Windows-Sysmon/Operational" -MaxEvents 5


   ProviderName: Microsoft-Windows-Sysmon

TimeCreated                      Id LevelDisplayName Message
-----------                      -- ---------------- -------
9/14/2026 1:51:49 AM             11 Information      File created:...
9/14/2026 1:51:49 AM              1 Information      Process Create:...
9/14/2026 1:51:47 AM              1 Information      Process Create:...
9/14/2026 1:46:21 AM              3 Information      Network connection detected:...
9/14/2026 1:35:35 AM             22 Information      Dns query:...


PS C:\Users\Someone Unknown\Desktop\Sysmon\Sysmon>
```

Let's check out ID 1 and I see that -> CommandLine: "C:\WINDOWS\System32\WindowsPowerShell\v1.0\powershell.exe" -enc VwByAGkAdABlAC0ASABvAHMAdAAgAGgAaQA=

Now let's decode using PowerShell:

```
[System.Text.Encoding]::Unicode.GetString([Convert]::FromBase64String("VwByAGkAdABlAC0ASABvAHMAdAAgAGgAaQA="))
```

What is Unicode? Unicode is a giant list that gives every letter, number, and symbol its own code, so computers everywhere show text the same way.

Results:

```
PS C:\Users\Someone Unknown\Desktop\Sysmon\Sysmon> [System.Text.Encoding]::Unicode.GetString([Convert]::FromBase64String("VwByAGkAdABlAC0ASABvAHMAdAAgAGgAaQA="))
Write-Host hi
PS C:\Users\Someone Unknown\Desktop\Sysmon\Sysmon>
```

## Filter It Better

```
Get-WinEvent -LogName "Microsoft-Windows-Sysmon/Operational" -FilterXPath "*[System[EventID=1]]" -MaxEvents 10 | Where-Object { $_.Message -like "*-enc*" } | Format-List
```

Now this is what will just straight up show what is encoded, rather than us going one by one.

Where-Object = a filter. Keeps only items that pass a test.
{ } = the test goes inside these braces.
$_ = "the current item" being checked (one event).
.Message = the field on that event you want to look at.
-like = compare with wildcard matching.
"*-enc*" = the pattern. * means "anything," so this matches any message containing -enc anywhere.

Basically, Where-Object is running a test to see if the item has that key:value pair.

# Conclusion

So far we covered how to download the Sysmon CLI from Sysinternals, then unzip it in a folder. Then we ran Sysmon and studied the processes. Then we made a fake encoded string in PowerShell, and the Sysmon CLI caught it, and we were able to read it. This scenario satisfies NIST 800-171 3.3.1 by creating and retaining detailed process and network audit records at the system level.
