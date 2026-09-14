# Sysmon Deployment - Looking at the deep Windows process & network visibility for threat data gathering (NIST 800-171: 3.3.1)

---

My Windows VM already gathers baseline process creation, so we do not need to mess with audit policy. So we can just go ahead and download Sysmon (System Monitor). We can download it straight from PowerShell into a folder from Windows Sysinternals.

PowerShell:

```powershell
Invoke-WebRequest -Uri "https://download.sysinternals.com/files/Sysmon.zip" -OutFile "Sysmon.zip"
Expand-Archive Sysmon.zip -DestinationPath Sysmon
```

-Uri - Uniform Resource Locator. Also if you ever want to change directory and the username has space in it, like -> Random User -> you can use -> cd "Location" and it will not cause any error issues in PowerShell.

Expand-Archive - Unzips a zip file. 
Sysmon.zip - The source zip file.
-DestinationPath - The folder to dump all that Sysmon.zip items or files & if the Sysmon doesn't exist it will create one.

---

Also Sysmon is CLI program. Open PowerShell and CD into the Sysmon folder and run it there -> .\Sysmon64.exe
