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









