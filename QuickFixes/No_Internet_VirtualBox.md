# Windows 11 VM had no internet (VirtualBox)
### Small problem I ran into while I was trying to download a file through PowerShell
---

I do not have working internet. Well, Ping 8.8.8.8 did not work. Ping timed out. So it was not just DNS, the VM had no route out at all. So typed in `ipconfig` and it's sitting on 192.168.100.10 and the gateway 192.168.100.1. Which is wrong for VirtualBox NAT, because NAT will hand out usually 10.0.2.x and its gateway 10.0.2.2. I also pinged the gateway which timed out too, so nothing was being routed there. I knew it was not DNS because 8.8.8.8 is itself raw IP and if that fails, the problem is the connection itself, not DNS.

---

Why it was broken?
1. The adapter was on a network that was not routing to the internet
2. The IP was set to static, which I did quite a while ago to test with pfSense. Windows was still using that frozen old address and did not pull a new one, and that is why `ipconfig /release` and `/renew` did not work either. It said -> "no adapter is in the state permissible for this operation." Those commands only works on DHCP.

---

`ipconfig /release` drops your current DHCP address; `/renew` asks the DHCP server for a fresh one.

---

DHCP is the service that automatically gives the machine/OS an IP and gateway/DNS instead of us typing it by hand.

---

So now the main issue was the adapter was pointing towards a dead network with no router & static IP made it worse by locking onto the dead address so it couldn't recover. Static itself does not cause "no internet". Static IP only works if the number we set (gateway, subnet, DNS) points towards a real router. Mine was pointed at 192.168.100.1, which nothing was routing, so there was no way out.

---

# FIX
1. Make sure the adapter is on NAT which has to be on Adapter 1.
2. Then through PowerShell turn the adapter off static and turn on automatic (DHCP).


```
netsh interface ip set address name="Ethernet" source=dhcp
netsh interface ip set dns name="Ethernet" source=dhcp
```

-netsh - network shell
-interface - digital plug/port that let's our computer connect to the network.

FIRST -> Get an automatic IP from our gateway router for the Ethernet interface.
SECOND -> Get the DNS server settings automatically from the router.

Why the fix works: NAT gives the VM a real router (VirtualBox runs one behind the scenes).DHCP just tells windows to ask for one from there.Once both were set, Windows pulled a fresh 10.0.2.x lease and traffic flows. Internet back.
