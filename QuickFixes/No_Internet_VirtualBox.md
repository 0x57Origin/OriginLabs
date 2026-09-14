# Windows 11 VM had no internet (VirtualBox)
### Small problem I ran into while I was trying to download a file through powersehll
---

I do not have working internet. Well Ping 8.8.8.8 did not work. Ping timed out. So it was not just DNS, the VM had no route out at all. So typed in ```ifconfig``` and it's sitting on 192.168.100.10 and the gateway 192.168.100.1. Which is wrong for VirtualBox NAT, because NAT will hand out usually 10.0.2.x and it's gateway 10.0.2.2. I also pinged the gateway which timed out too, so nothing was being routed there. I knew it was not DNS because 8.8.8.8 is itself raw Ip and if that fails , the problem is the connection itself, not DNS.

---

Why it was broken?
1. The adapter was on a network that was not routing to the internet
2. The IP was set to static, which I did quite awhile ago to test with PFSENSE. Windows was still using that frozen old address and did not pull a new one, and that is why ipconfig /release and /renew did not work either. It said -> "no adapter is in the state permissible for this operation." Those commands only works on DHCP.
---

ipconfig /release drops your current DHCP address; /renew asks the DHCP server for a fresh one.
---

DHCP is the service that automatically gives the machine/OS an IP and gateway/DNS instead of us typing it by hand.

---

So now the main issue was the adatper was pointing towards a dead network with no router & static IP made it worse by locking onto the dead address so it couldnt recover. Static it self does not cause "no internet". Static IP only works if the number we set (gateway, subnet, DNS) points towards a real router. Mine was pointed at 192.168.100.1, which nothing was routing, so there was no way out.

