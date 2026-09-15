Make sure to run as Administrator:

```powershell
# 1. Install with a config file
.\Sysmon64.exe -accepteula -i sysmonconfig.xml

# 2. Check it's running
Get-Service Sysmon64

# 3. See the current config
.\Sysmon64.exe -c

# 4. Update to a new config (no reinstall)
.\Sysmon64.exe -c newconfig.xml

# 5. Uninstall
.\Sysmon64.exe -u

# 6. Show the 10 newest Sysmon events
Get-WinEvent -LogName "Microsoft-Windows-Sysmon/Operational" -MaxEvents 10

# 7. Pull one event type (faster than Where-Object)
Get-WinEvent -FilterHashtable @{LogName="Microsoft-Windows-Sysmon/Operational"; Id=3} -MaxEvents 5

# 8. Search for an IP, domain, or process name
Get-WinEvent -FilterHashtable @{LogName="Microsoft-Windows-Sysmon/Operational"; Id=3} | Where-Object {$_.Message -match "1.1.1.1"} | Select-Object -First 1 -ExpandProperty Message

# 9. Open it in Event Viewer
eventvwr.msc
# Path: Applications and Services Logs > Microsoft > Windows > Sysmon > Operational
```

**Event IDs worth memorizing:**

| ID | What it logs |
|----|-------------|
| 1 | Process created |
| 3 | Network connection |
| 11 | File created |
| 13 | Registry value set |
| 22 | DNS query |

Sysmon with no config logs very little.
