# Is my windows SSH Exposed?
---
We will confirm if your window's SSH port is reachable via the public internet & we will shut it if it is.
---
First let's see find our public ip.
```
curl ifconfig.me
```
I will use example public IP -> 12.98.23.22
---
Let's go to Shodan first and see if our port is exposed or not because Shodan searches the whole internet and looks for it.
```
https://www.shodan.io/host/12.98.23.22

404: Not Found
Note:
No information available for 12.157.79.124
```
Well that is a very good sign but also remember Shodan is also an old photo if you were not in the photo does not mean you were not there when the photo was taken. Let's test our system live.

Let's knock on port 22 from outside our network. Your own PC can't truly test itself from outside. So we use a website that probes you from its servers.

```
https://www.yougetsignal.com/tools/open-ports/
```

Change the port to 22. After I checked it told me that it got filtered, meaning it's not open or closed it just don't want to talk to any knocking connections. Nobody outside can reach your SSH. Firewall dropping the packet silently.

Final Verdict: Windows Root or any SSH is not exposed.

