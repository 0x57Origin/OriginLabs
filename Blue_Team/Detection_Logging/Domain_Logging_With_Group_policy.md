# Domain Logging With Group Policy
---
We will be pushing audit policy, command-line logging, and bigger log sizes through a GPO, deploy Sysmon through a GPO, set up Windows Event Forwarding to one collector, then we will prove events from the domain joined host are landing.
---
Objective:
We will build a small domain, Use Group policy from there to push audit settings and Sysmon.  Forward those events to one collector and prove they will land. We will put everything on HOST only network. In real life just use the LAN.
In Windows Server the IP is 192.168.56.104 and is the DC = Domain Controller. In the server's shell just type in 8 and it will give you the network information. Windows 11 VM IP =  192.168.56.101. In Windows Server if you type ```sconfig``` it will take you to the main menu. sconfig = server config.
Now in DC type 15 and type:
```netsh advfirewall firewall add rule name="Allow Ping" protocol=icmpv4:8,any dir=in action=allow``` 
Which will allow it to get pinged and firewall won't block it. And make sure to ping from both sides. We can use the same command on Windows 11 VM too but if that does not work:
```
New-NetFirewallRule -DisplayName "Allow Ping" -Protocol ICMPv4 -IcmpType 8 -Direction Inbound -Action Allow
```
Now let's name the DC to DC01 by typing 2 in the ```sconfig``` and then restart. Then type 15 and it will lead you to shell and type hostname and it will say DC01. I will not keep saying the same thing over and over just type 15 to get the shell/terminal & type ```sconfig``` to get to the main menu.

## Install Active Directory: Make it a real Domain Controller. 
Difference: AD is the directory of users, computers and policy. A DC is the server that hosts and runs that directory.
Now in DC01: You can type in PowerShell in normal shell after typing 15 in server.
```
Install-WindowsFeature AD-Domain-Services, DNS -IncludeManagementTools
```
Now we will install ADDSForest: AD DS = Active Directory Domain Services. The Windows service that stores users, computers, and Group Policy. Forest = A new AD world. The first domain we will create.
But before what I need to explain workgroup: Every PC is alone. Windows Server and Windows 11 do not share users. That is what we have right now.  Domain = One boss computer keeps the user list. Other PCs join it and use those users. That boss is the DC. 
What we have installed: We installed the software for AD and DNS. Like buying the toolbox but we have not built the house yet.  So when we do install ```ADDSForest``` that will build the domain named ```origin.lab``` and turn DC01 into that boss computer. 
Install-ADDSForest = install AD DS and create that first domain on this server. It will ask for a password. Server will restart after that.
```
Install-ADDSForest -DomainName "origin.lab" -InstallDns -Force
```
Now on Windows 11 VM:
```
Press Win + R
Type ncpa.cpl and hit Enter
Right-click the adapter that has 192.168.56.101
Properties
Click Internet Protocol Version 4 (TCP/IPv4)
Properties
Select Use the following DNS server addresses
Preferred DNS: 192.168.56.104
Leave Alternate blank
OK, OK
```

Now Ping it and you will get replies and do ```ipconfig /all```:
```
Windows IP Configuration

   Host Name . . . . . . . . . . . . : DESKTOP-APE8POJ
   Primary Dns Suffix  . . . . . . . :
   Node Type . . . . . . . . . . . . : Hybrid
   IP Routing Enabled. . . . . . . . : No
   WINS Proxy Enabled. . . . . . . . : No

Ethernet adapter Ethernet:

   Connection-specific DNS Suffix  . :
   Description . . . . . . . . . . . : Intel(R) PRO/1000 MT Desktop Adapter
   Physical Address. . . . . . . . . : 08-00-27-C8-81-90
   DHCP Enabled. . . . . . . . . . . : Yes
   Autoconfiguration Enabled . . . . : Yes
   Link-local IPv6 Address . . . . . : fe80::f091:696d:3436:7584%13(Preferred)
   IPv4 Address. . . . . . . . . . . : 192.168.56.101(Preferred)
   Subnet Mask . . . . . . . . . . . : 255.255.255.0
   Lease Obtained. . . . . . . . . . : Sunday, September 20, 2026 12:05:17 PM
   Lease Expires . . . . . . . . . . : Sunday, September 20, 2026 1:45:20 PM
   Default Gateway . . . . . . . . . :
   DHCP Server . . . . . . . . . . . : 192.168.56.100
   DHCPv6 IAID . . . . . . . . . . . : 84410407
   DHCPv6 Client DUID. . . . . . . . : 00-01-00-01-32-22-9A-B9-08-00-27-C8-81-90
   DNS Servers . . . . . . . . . . . : 192.168.56.104
   NetBIOS over Tcpip. . . . . . . . : Enabled
PS C:\WINDOWS\system32>
```
DNS is already 192.168.56.104, in the same terminal do this:
```
ping origin.lab
nslookup origin.lab
```
Results:
```
PS C:\WINDOWS\system32> ping origin.lab

Pinging origin.lab [192.168.56.104] with 32 bytes of data:
Reply from 192.168.56.104: bytes=32 time=1ms TTL=128
Reply from 192.168.56.104: bytes=32 time=1ms TTL=128
Reply from 192.168.56.104: bytes=32 time=1ms TTL=128
Reply from 192.168.56.104: bytes=32 time=6ms TTL=128

Ping statistics for 192.168.56.104:
    Packets: Sent = 4, Received = 4, Lost = 0 (0% loss),
Approximate round trip times in milli-seconds:
    Minimum = 1ms, Maximum = 6ms, Average = 2ms
PS C:\WINDOWS\system32> nslookup origin.lab
DNS request timed out.
    timeout was 2 seconds.
Server:  UnKnown
Address:  192.168.56.104

Name:    origin.lab
Address:  192.168.56.104

PS C:\WINDOWS\system32>
```
nslookup = Name Server Lookup is a command-line tool used to query Domain Name System (DNS) servers to find IP addresses, domain names, and specific DNS records.
Domain DNS is working. origin.lab is 192.168.56.104
Now we can join it through Windows, same PowerShell:
```
Add-Computer -DomainName "origin.lab" -Restart
```
It will ask for username which wil be = origin\Administrator and the password will be of the server, meaning when we first installed the Windows Server on VM that password not the password from Install-ADDSForest. Very important, remember that. Password is not the DSRM / Safe Mode password from Install-ADDSForest. It is the Administrator password on the Server from when you first installed Windows Server.
Now if we put whoami in the windows 11 terminal it will still say the same name, meaning we are still logged in as a local user = desktop-ape8poj\someone unknown. So we need to check if we even joined:
```
systeminfo | findstr /B /C:"Domain" /C:"System"
```
If Domain is ```origin.lab```. Then we sign out and login screen click Other user.  User: origin\Administrator & Password: Server Administrator password...
Just to show you mine worked:
```
PS C:\WINDOWS\system32> whoami
desktop-ape8poj\someone unknown
PS C:\WINDOWS\system32> systeminfo | findstr /B /C:"Domain" /C:"System"
System Boot Time:              9/20/2026, 1:54:45 PM
System Manufacturer:           innotek GmbH
System Model:                  VirtualBox
System Type:                   x64-based PC
System Directory:              C:\WINDOWS\system32
System Locale:                 en-us;English (United States)
Domain:                        origin.lab
PS C:\WINDOWS\system32>
```
Now let us sign out & then login to the new account. Now:
```
Windows PowerShell
Copyright (C) Microsoft Corporation. All rights reserved.

Install the latest PowerShell for new features and improvements! https://aka.ms/PSWindows

PS C:\WINDOWS\system32> whoami
origin\administrator
PS C:\WINDOWS\system32>
```

Now the domain join is done. Next is Group Policy.
Windows Server: DC01
```
Import-Module GroupPolicy
New-GPO -Name "Origin-Logging" | New-GPLink -Target "DC=origin,DC=lab"                
Get-GPO -Name "Origin-Logging"
```
In that 2nd command DC means Domain Component. It is how Active Directory writes the domain name. origin.lab has 2 parts so the path is -> DC=origin,DC=lab -> means: attach this GPO to the whole origin.lab domain.
If you see the GPO name and an ID, it worked.
<img width="1015" height="597" alt="image" src="https://github.com/user-attachments/assets/01af339e-ecf9-4767-bc14-e5ad8f62c8a4" />

On DC01, add command-line logging to the GPO. One command only: GROUP POLICY OBJECT -  It turns on the command-line text inside process-creation logs. Without it , Windows will say -> PowerShell started. With it, the log can also show what was typed after powershell.exe like - enc. That is the 4688 Sysmon-style detail we want from Group Policy. The command will write that switch into the GPO so windows gets it after gpupdate.
Run this on DC01, in PowerShell:
```
Set-GPRegistryValue -Name "Origin-Logging" -Key "HKLM\Software\Microsoft\Windows\CurrentVersion\Policies\System\Audit" -ValueName "ProcessCreationIncludeCmdLine_Enabled" -Type DWord -Value 1
```
<img width="1007" height="566" alt="image" src="https://github.com/user-attachments/assets/5d2d856a-b7a5-46d0-a942-5d43eac051de" />

So now we know it worked because Computer Version went to 1. The GPO has that setting.

Next command which you still put in DC01: That makes the Security log bigger so events do not roll over as fast. Basically, when the security log fills up, Windows will start deleting old events to make room, but we will make it bigger so it holds more.
```
Set-GPRegistryValue -Name "Origin-Logging" -Key "HKLM\Software\Policies\Microsoft\Windows\EventLog\Security" -ValueName "MaxSize" -Type DWord -Value 262144
```
<img width="985" height="547" alt="image" src="https://github.com/user-attachments/assets/f3f6d95b-a5fa-4e85-b7e4-61352506c301" />

Next command: Old Windows auditing is vague. This switch turns on the detailed audit list so you can see logs like process creation properly. 
```
Set-GPRegistryValue -Name "Origin-Logging" -Key "HKLM\SYSTEM\CurrentControlSet\Control\Lsa" -ValueName "SCENoApplyLegacyAuditPolicy" -Type DWord -Value 1
```
<img width="1012" height="562" alt="image" src="https://github.com/user-attachments/assets/ad49406a-4d42-45d1-8ac8-c3153b3d5e46" />
Worked. Version is 3.

Now go to Windows 11 VM, logged in as origin\administrator and run:
```
gpupdate /force
```
That pulls Origin-Logging into the PC. 

# QUICK NOTE: The logs, they stay on the PC's disk, so it is permanent and we can go back and check those logs anytime.

But anyway the result is:
```
PS C:\WINDOWS\system32> gpupdate /force
Updating policy...

Computer policy could not be updated successfully. The following errors were encountered:

The processing of Group Policy failed because of lack of network connectivity to a domain controller. This may be a transient condition. A success message would be generated once the machine gets connected to the domain controller and Group Policy has successfully processed. If you do not see a success message for several hours, then contact your administrator.
User Policy could not be updated successfully. The following errors were encountered:

The processing of Group Policy failed because of lack of network connectivity to a domain controller. This may be a transient condition. A success message would be generated once the machine gets connected to the domain controller and Group Policy has successfully processed. If you do not see a success message for several hours, then contact your administrator.

To diagnose the failure, review the event log or run GPRESULT /H GPReport.html from the command line to access information about Group Policy results.
PS C:\WINDOWS\system32>
```

Soloution: This happend because of the VM and it' settings and a lot of times it will just break it will most likely not happen on a real corporate settings and if it does good luck.
I pinged the server from windows 11 VM and it did reply back but when I did ipconfig /all it shows DNS as those fec0... addresses, not 192.168.56.104. VirtualBox DHCP 192.168.56.100 wiped our DNS. 
So now on Windows 11, ```ncpa.cpl``` again: Network Control Panel Applet
```
That 192.168.56.101 adapter
IPv4 Properties
Use the following DNS: 192.168.56.104
OK
```
Then  I pinged ```origin.lab``` again and it is now fixed.
Results:
```
PS C:\WINDOWS\system32> gpupdate /force
Updating policy...

Computer Policy update has completed successfully.
User Policy update has completed successfully.

PS C:\WINDOWS\system32>
```
Policy applied so now on Windows 11 check if it landed:
```
gpresult /r
```
Make sure to look under Computer Settings for Origin-Logging.

Result:
```
COMPUTER SETTINGS
------------------
    CN=DESKTOP-APE8POJ,CN=Computers,DC=origin,DC=lab
    Last time Group Policy was applied: 9/20/2026 at 5:33:12 PM
    Group Policy was applied from:      DC01.origin.lab
    Group Policy slow link threshold:   500 kbps
    Domain Name:                        ORIGIN
    Domain Type:                        Windows 2008 or later

    Applied Group Policy Objects
    -----------------------------
        Default Domain Policy
        Origin-Logging -> RIGHT HERE********************************

    The following GPOs were not applied because they were filtered out
    -------------------------------------------------------------------
        Local Group Policy
            Filtering:  Not Applied (Empty)

    The computer is a part of the following security groups
    -------------------------------------------------------
        BUILTIN\Administrators
        Everyone
        BUILTIN\Users
        NT AUTHORITY\NETWORK
        NT AUTHORITY\Authenticated Users
        This Organization
        DESKTOP-APE8POJ$
        Domain Computers
        Authentication authority asserted identity
        System Mandatory Level
```
Now let's check the registry value:
```
Get-ItemProperty "HKLM:\Software\Microsoft\Windows\CurrentVersion\Policies\System\Audit"
```
Mine shows it look:
```
ProcessCreationIncludeCmdLine_Enabled : 1 -> This is it right here.
PSPath                                : Microsoft.PowerShell.Core\Registry::HKEY_LOCAL_MACHINE\Software\Microsoft\Windows\CurrentVersion\Policies\System\Audit
PSParentPath                          : Microsoft.PowerShell.Core\Registry::HKEY_LOCAL_MACHINE\Software\Microsoft\Windows\CurrentVersion\Policies\System
PSChildName                           : Audit
PSDrive                               : HKLM
PSProvider                            : Microsoft.PowerShell.Core\Registry
```
And right here the GPO chunk is done. Next we will deploy Sysmon through a GPO. How? We will put Sysmon on a share folder that Win11 can read. So Sysmon does not come from Group Policy itself. GPO only points the PC at files and it says hey Install this. We will put Sysmon64.exe and the config XML in a folder on DC01 and share that folder as \\DC01\Sysmon. Windows 11 VM will read from the share and install Sysmon.

on DC01:
```
New-Item -Path C:\Share\Sysmon -ItemType Directory -Force
New-SmbShare -Name Sysmon -Path C:\Share\Sysmon -ReadAccess "Everyone"
```
After the folder and it is shared we will copy sysmon folder that we previously downlaoded in windows 11 VM and we will copy that into \\DC01\Sysmon.

Well for this lab I will download Sysmon again in Windows 11 VM. HOW TO IDOWNLOAD SYSMON:
```
Invoke-WebRequest -Uri "https://download.sysinternals.com/files/Sysmon.zip" -OutFile "$env:USERPROFILE\Downloads\Sysmon.zip"
Expand-Archive "$env:USERPROFILE\Downloads\Sysmon.zip" -DestinationPath "$env:USERPROFILE\Downloads\Sysmon" -Force
```
**After that we will move it in the C:\Sysmon folder okay.**

Now let's give the share folder write access for ORIGIN\Administrator so we can over our files: ON DC01
```
Grant-SmbShareAccess -Name Sysmon -AccountName "ORIGIN\Administrator" -AccessRight Full -Force
icacls C:\Share\Sysmon /grant "ORIGIN\Administrator:(OI)(CI)F"
```
<img width="1027" height="530" alt="image" src="https://github.com/user-attachments/assets/80933858-a4a3-461e-9b73-30da5626f83f" />
icacls = Integrity Control Access Control List sets who can use that folder on disk. This line gives origin\Administrator full control of C:\Share\Sysmon and everything inside it so the copy can write files.

Now on Windows 11 VM: Let's copy the Sysmon files to the share.
```
Copy-Item "C:\Sysmon\*" -Destination "\\DC01\Sysmon\" -Recurse
dir \\DC01\Sysmon
```

Results:
```
    Directory: \\DC01\Sysmon


Mode                 LastWriteTime         Length Name
----                 -------------         ------ ----
-a----         9/10/2026   3:03 PM           7490 Eula.txt
-a----         9/10/2026   3:06 PM        6162240 Sysmon.exe
-a----         9/10/2026   3:06 PM        3203368 Sysmon64.exe
-a----         9/10/2026   3:06 PM        3138824 Sysmon64a.exe
```
Share folder is good & Sysmon64.exe is there. But as you can see no .xml file or config file.
So what we can do right now is on Windows 11 VM. let's download the sysmonconfig-export.xml from: https://github.com/SwiftOnSecurity/sysmon-config and copy that over to \\DC01\Sysmon\
So in C:\Sysmon we will download it there or we can just copy and paste what is in there over the GitHub and call it like mainconfig.xml. After that copy that to the DC. 

Now we will again copy that over to server share folder:
```
Copy-Item "C:\Sysmon\*" -Destination "\\DC01\Sysmon\" -Recurse
dir \\DC01\Sysmon
```
And since it worked now we can go ahead and make the install script on DC01:
```
@"
@echo off
sc query Sysmon64 >nul 2>&1 && exit /b 0
\\DC01\Sysmon\Sysmon64.exe -accepteula -i \\DC01\Sysmon\mainconfig.xml
"@ | Set-Content C:\Share\Sysmon\install-sysmon.bat
```
Breakdown:
@" = Start of a multi line string and everything until "@ is file content.
@echo off = Do not print every command when the scripts runs.
sc = is the Service Control. Windows tool for services.
query Sysmon64 = it askes like is there a service named Sysmon64?
>nul =  it hides the normal output.
2>&1 = Hides the error output too. I don't want that text on screen.
&& = Only if that worked.
exit /b 0 =  Stop the script, success.
\\DC01\Sysmon\Sysmon64.exe -accepteula -i \\DC01\Sysmon\mainconfig.xml = This is us installing the Sysmon with the config XML file.
"@ | Set-Content C:\Share\Sysmon\install-sysmon.bat = "@ ends the text block and | sends that text into the next command & everything after | just writes text to that file. So PowerShell is not running Sysmon here. It just saves the .bat file on the DC server.

Then check ```dir C:\Share\Sysmon``` and you will see the file exists on there. Now attach the script to the GPO. On DC01:































