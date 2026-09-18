# How to develop a network diagram

---

Well now run NMAP now to find out every live host on the network.

```
nmap -sn 192.168.56.0/24 -oN hosts_found.txt
```
BREAKDOWN: 

```-sn = no port scan , it will only do host discovery.```

RESULTS:

```
Starting Nmap 7.99 ( https://nmap.org ) at 2026-09-18 01:31 -0400
Nmap scan report for 192.168.56.100
Host is up (0.00034s latency).
MAC Address: 08:00:27:0F:5A:35 (Oracle VirtualBox virtual NIC)
Nmap scan report for 192.168.56.101
Host is up (0.00089s latency).
MAC Address: 08:00:27:C8:81:90 (Oracle VirtualBox virtual NIC)
Nmap scan report for 192.168.56.102
Host is up (0.00036s latency).
MAC Address: 0A:00:27:00:00:03 (Unknown)
Nmap scan report for 192.168.56.103
Host is up.
Nmap done: 256 IP addresses (4 hosts up) scanned in 5.21 seconds
```

All 4 Addresses are up = .100, .101, .102, .103.

Now 103 is my address because because Nmap can't show a MAC for the machine it's scanning from & if you look closely 103 does not have a MAC address. Also Host is up is a sign because there was no latency.

So now we are down to 3. So run a service scan on the three real targets:

```
nmap -sV 192.168.56.100-102
```

RESULTS:

```
Starting Nmap 7.99 ( https://nmap.org ) at 2026-09-18 01:36 -0400
Nmap scan report for 192.168.56.100
Host is up (0.00043s latency).
All 1000 scanned ports on 192.168.56.100 are in ignored states.
Not shown: 1000 closed tcp ports (reset)
MAC Address: 08:00:27:0F:5A:35 (Oracle VirtualBox virtual NIC)

Nmap scan report for 192.168.56.101
Host is up (0.0024s latency).
Not shown: 994 filtered tcp ports (no-response)
PORT     STATE SERVICE       VERSION
135/tcp  open  msrpc         Microsoft Windows RPC
139/tcp  open  netbios-ssn   Microsoft Windows netbios-ssn
445/tcp  open  microsoft-ds?
3389/tcp open  ms-wbt-server
5357/tcp open  http          Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
5985/tcp open  http          Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
1 service unrecognized despite returning data. If you know the service/version, please submit the following fingerprint at https://nmap.org/cgi-bin/submit.cgi?new-service :
SF-Port3389-TCP:V=7.99%I=7%D=9/18%Time=6AACCDF9%P=x86_64-pc-linux-gnu%r(Te
SF:rminalServerCookie,13,"\x03\0\0\x13\x0e\xd0\0\0\x124\0\x02/\x08\0\x02\0
SF:\0\0");
MAC Address: 08:00:27:C8:81:90 (Oracle VirtualBox virtual NIC)
Service Info: OS: Windows; CPE: cpe:/o:microsoft:windows

Nmap scan report for 192.168.56.102
Host is up (0.00014s latency).
All 1000 scanned ports on 192.168.56.102 are in ignored states.
Not shown: 1000 filtered tcp ports (no-response)
MAC Address: 0A:00:27:00:00:03 (Unknown)

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 3 IP addresses (3 hosts up) scanned in 17.93 seconds
```

Breakdown:

On .100 all 1000 ports are closed. It does show a MAC address. Host is up. A host with no open services.

Let's just quickly figure out what .100 is:

```
sudo nmap -O 192.168.56.100
```
-O = Os.

Results:

```
Warning: OSScan results may be unreliable because we could not find at least 1 open and 1 closed port

That's just means it is junk and just ignore it.
```

Now from the previous scan ```nmap -sV 192.168.56.100-102``` we found that .101 is Windows 11 VM.

.102 ignored our Nmap scan. Firewalled.

Color logic:
- **Green** = My Machine (Kali)
- **Red** = .101, the only host with a real attack surface (SMB, RDP, WinRM)
- **Orange** = .102, alive but firewalled
- **Gray** = .100, alive but nothing reachable

<img width="2125" height="1292" alt="network-recon-map" src="https://github.com/user-attachments/assets/dfc269c0-d5ff-418f-86cc-fd722547951a" />

