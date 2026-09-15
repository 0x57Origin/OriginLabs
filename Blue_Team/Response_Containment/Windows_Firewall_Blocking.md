# Windows Firewall blocking.
---

Let's block an outbound connection via firewall. We need to confirm it via firewall log & Sysmon.

---
Target Scope: Cloudflare's 1.1.1.1 on port 443

For Windows Firewall we will be using TCP test to see if we can reach the target. We will test the same channel that we will block later on , meaning TCP. 

PowerShell:
```
Test-NetConnection 1.1.1.1 -Port 443

ComputerName     : 1.1.1.
RemoteAddress    : 1.1.1.1
RemotePort       : 443
InterfaceAlias   : Ethernet
SourceAddress    : 10.0.2.15
TcpTestSucceeded : True -> This is what we want to see. 
```

Well now let's block it. 

```
New-NetFirewallRule -DisplayName "Block 1.1.1.1 443" -Direction Outbound -Action Block -RemoteAddress 1.1.1.1 -RemotePort 443 -Protocol TCP


Name                          : {679db5cb-f50e-447b-b13d-cc983610889e}
DisplayName                   : Block 1.1.1.1 443
Description                   :
DisplayGroup                  :
Group                         :
Enabled                       : True
Profile                       : Any
Platform                      : {}
Direction                     : Outbound
Action                        : Block
EdgeTraversalPolicy           : Block
LooseSourceMapping            : False
LocalOnlyMapping              : False
Owner                         :
PrimaryStatus                 : OK
Status                        : The rule was parsed successfully from the store. (65536)
EnforcementStatus             : NotApplicable
PolicyStoreSource             : PersistentStore
PolicyStoreSourceType         : Local
RemoteDynamicKeywordAddresses : {}
PolicyAppId                   :
PackageFamilyName             :
```

Rerun the test net connection and let's see what happens.

```
Test-NetConnection 1.1.1.1 -Port 443

ComputerName     : 1.1.1.1
RemoteAddress    : 1.1.1.1
RemotePort       : 443
InterfaceAlias   : Ethernet
SourceAddress    : 10.0.2.15
TcpTestSucceeded : True

For some reason it's not working.
```

Alright I found it the Windows Firewall Profile was turned off due to a previous lab I was working on. Here is the command to see it.
```
Get-NetFirewallProfile | Select-Object Name, Enabled

Name    Enabled
----    -------
Domain    False
Private   False
Public    False
```

Let's turn it on:
```
Set-NetFirewallProfile -Profile Domain,Private,Public -Enabled True
```

After that if you want to confirm if it worked or not we can use the Get-NetFirewallProfile again to see if it worked.

```
Test-NetConnection 1.1.1.1 -Port 443
WARNING: TCP connect to (1.1.1.1 : 443) failed


ComputerName           : 1.1.1.1
RemoteAddress          : 1.1.1.1
RemotePort             : 443
InterfaceAlias         : Ethernet
SourceAddress          : 10.0.2.15
PingSucceeded          : True
PingReplyDetails (RTT) : 297 ms
TcpTestSucceeded       : False
```

But for me it is working. 

# Now let's check the Firewall Loggin

Let's first configure the firewall to record every blocked packet into a text log file and it's called = pfirewall.log. Very important or firewall won't keep a record of it. **It is not turned on by default**.

Command:
```
Set-NetFirewallProfile -LogBlocked True -LogFileName "C:\Windows\System32\LogFiles\Firewall\pfirewall.log"
```

Now we will test connection again ->  Test-NetConnection 1.1.1.1 -Port 443

After that we will read the log file using notepad. 

```
notepad C:\Windows\System32\LogFiles\Firewall\pfirewall.log

#Version: 1.5
#Software: Microsoft Windows Firewall
#Time Format: Local
#Fields: date time action protocol src-ip dst-ip src-port dst-port size tcpflags tcpsyn tcpack tcpwin icmptype icmpcode info path pid

2026-09-14 21:48:01 DROP TCP 10.0.2.15 1.1.1.1 63033 443 0 - 0 0 0 - - - SEND 4696    ->>> This is it we found it.
```

Now let's confirm using Sysmon. Always remember Sysmon Event ID 3 is the Network Connection event, which logs when a process opens a TCP/UDP connection. Let's check the event now.

```
Get-WinEvent -LogName "Microsoft-Windows-Sysmon/Operational" | Where-Object {$_.Id -eq 3 -and $_.Message -match "1.1.1.1"} | Select-Object -First 1 -ExpandProperty Message


Network connection detected:
RuleName: -
UtcTime: 2026-09-15 02:23:25.239
ProcessGuid: {c5100ea7-aede-6aa7-d501-000000001100}
ProcessId: 4696
Image: C:\Windows\System32\WindowsPowerShell\v1.0\powershell.exe
User: DESKTOP-APE8POJ\Someone Unknown
Protocol: tcp
Initiated: true
SourceIsIpv6: false
SourceIp: 10.0.2.15
SourceHostname: DESKTOP-APE8POJ
SourcePort: 63031
SourcePortName: -
DestinationIsIpv6: false
DestinationIp: 1.1.1.1
DestinationHostname: one.one.one.one
DestinationPort: 443
DestinationPortName: https

Image confirms it was done via powershell.exe
```

It logged it in Sysmon. Sysmon logs the initial attempt but remember Sysmon did not record it was success or not but our firewall log did.

Explanation of the command:

$_ = Represents the current log entry.

.Id = Access the specific ID. Which is 3 for Sysmon here.

-eq 3 =  Equals 3. Filtering the logs to keep only Sysmon Event ID 3 (Network Connection events).

.Message = Access the text body of the log.

-match "1.1.1.1" = Filters the logs to keep only the entries where the text body contains the IP address 1.1.1.1

That is all for this lab. 







