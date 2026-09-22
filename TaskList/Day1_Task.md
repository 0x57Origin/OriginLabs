# Day 1: Commands That Run Against Your Own Machine Only

Nothing here touches another host. Every command shows you something 
about the box you were handed. Network scanning waits for written authorization.

## Who Am I / What Am I On
whoami
whoami /priv                          # your privileges (see if you're admin)
hostname
[System.Environment]::OSVersion.Version   # exact Windows build

## Patch Level
Get-HotFix                            # every patch installed, with dates
Get-HotFix | Sort-Object InstalledOn -Descending | Select-Object -First 10

## What's Running (your process hunter, live)
Get-Process
Get-CimInstance Win32_Process | Select-Object ProcessId, Name, CommandLine
python main.py                        # run your own command-line hunter on this box

## What's Listening on the Network (your own machine's ports)
netstat -ano                          # every connection + owning PID
Get-NetTCPConnection -State Listen     # just what's listening

## Services and Autoruns (where persistence hides)
Get-Service | Where-Object {$_.Status -eq "Running"}
Get-CimInstance Win32_StartupCommand   # what auto-starts

## Local Users and Admins (who can log into this box)
Get-LocalUser
Get-LocalGroupMember Administrators     # who's local admin

## Logging Posture (is this box even auditing?)
auditpol /get /category:*              # what Windows is auditing
Get-WinEvent -ListLog Security | Select-Object LogName, MaximumSizeInBytes, RecordCount
Get-Service Sysmon*                    # is Sysmon installed?

## Firewall State
Get-NetFirewallProfile | Select-Object Name, Enabled
